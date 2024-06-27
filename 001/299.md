Clumsy Clay Mole

Medium

# `DepositWrapper.deposit()`: incorrect handling of `steth` token transfer

## Summary

`DepositWrapper.deposit()` doesn't handle the transferred `steth` tokens correctly (as the amount transferred will be less by 1 to 2 wei) resulting in disabling the support of this token in the `deposit()` function.

## Vulnerability Detail

- `DepositWrapper.deposit()` function allows depositing `wstETH` in vaults that have only one underlying token which is the `wstETH`.

- This function enables users to provide any of `stETH`, `wETH` or native `ETH` and then convert the provided non `wstETH` to `wstETH` token:

```js
// @notice : `DepositWrapper.deposit()` function
   function deposit(
        address to,
        address token,
        uint256 amount,
        uint256 minLpAmount,
        uint256 deadline
    ) external payable returns (uint256 lpAmount) {
       //....

        if (token == steth) {
            IERC20(steth).safeTransferFrom(sender, wrapper, amount);
            amount = _stethToWsteth(amount);
        } else if (token == weth) {
            IERC20(weth).safeTransferFrom(sender, wrapper, amount);
            amount = _wethToWsteth(amount);
        } else if (token == address(0)) {
            if (msg.value != amount) revert InvalidAmount();
            amount = _ethToWsteth(amount);
        } else if (wsteth == token) {
            IERC20(wsteth).safeTransferFrom(sender, wrapper, amount);
        } else revert InvalidToken();

        //.....
    }
```

where:

```js
    function _stethToWsteth(uint256 amount) private returns (uint256) {
        IERC20(steth).safeIncreaseAllowance(wsteth, amount);
        IWSteth(wsteth).wrap(amount);
        return IERC20(wsteth).balanceOf(address(this));
    }
```

## Impact

- But this function doesn't handle `stETH` token conversion correctly, as the `IERC20(steth).safeTransferFrom(sender, wrapper, amount);` will transfer 1-2 wei less than the transferred `amount`, which would result in reverting the consequent `_stethToWsteth()` that converts the `amount` provided but not the actual received which is less by 1-2 wei (insufficient contract balance).

- This is due to a [known rounding down issue](https://github.com/lidofinance/lido-dao/issues/442) in the `stETH` token contract that uses shares for tracking balances.

- So this will result in DoS of the `deposit()` function with this token, and would result in DoS of any other 3-rd party protocols interacting with this function directly.

## Code Snippet

https://github.com/sherlock-audit/2024-06-mellow/blob/26aa0445ec405a4ad637bddeeedec4efe1eba8d2/mellow-lrt/src/utils/DepositWrapper.sol#L55C9-L57C45

## Tool used

Manual Review

## Recommendation

Update `DepositWrapper.deposit()` function to account for the actual transferred `wstETH` token:

```diff
   function deposit(
        address to,
        address token,
        uint256 amount,
        uint256 minLpAmount,
        uint256 deadline
    ) external payable returns (uint256 lpAmount) {
       //....

        if (token == steth) {
            IERC20(steth).safeTransferFrom(sender, wrapper, amount);
-           amount = _stethToWsteth(amount);
+           amount = _stethToWsteth(IERC20(steth).balanceOf(address(this)));
        } else if (token == weth) {
            IERC20(weth).safeTransferFrom(sender, wrapper, amount);
            amount = _wethToWsteth(amount);
        } else if (token == address(0)) {
            if (msg.value != amount) revert InvalidAmount();
            amount = _ethToWsteth(amount);
        } else if (wsteth == token) {
            IERC20(wsteth).safeTransferFrom(sender, wrapper, amount);
        } else revert InvalidToken();

        //.....
    }
```