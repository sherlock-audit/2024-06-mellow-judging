Clumsy Clay Mole

Medium

# DoS in `SimpleDVTStakingStrategy.convertAndDeposit()`

## Summary

`SimpleDVTStakingStrategy` contract doesn't have a `receive()` function to receive the withdrawn `WETH`, resulting in a txn revert.

## Vulnerability Detail

`SimpleDVTStakingStrategy.convertAndDeposit()` function is supposed to be delegate called by the `StakingModule.convertAndDeposit()` (`StakingModule.convertAndDeposit()` logic is executed with contract `SimpleDVTStakingStrategy` storage) where its implemented logic is to rececive `wETH` to convert it to `wstETH` to be deposited into a `depositSecurityModule` contract:

```js
// @notice : this is `SimpleDVTStakingStrategy.convertAndDeposit()`
    function convertAndDeposit(
        uint256 amount,
        uint256 blockNumber,
        bytes32 blockHash,
        bytes32 depositRoot,
        uint256 nonce,
        bytes calldata depositCalldata,
        IDepositSecurityModule.Signature[] calldata sortedGuardianSignatures
    ) external returns (bool success) {
        (success, ) = vault.delegateCall(
            address(stakingModule),
            abi.encodeWithSelector(
                IStakingModule.convertAndDeposit.selector,
                amount,
                blockNumber,
                blockHash,
                depositRoot,
                nonce,
                depositCalldata,
                sortedGuardianSignatures
            )
        );
        emit ConvertAndDeposit(success, msg.sender);
    }
```

```js
// @notice : this is `StakingModule.convertAndDeposit()`
   function convertAndDeposit(
        uint256 amount,
        uint256 blockNumber,
        bytes32 blockHash,
        bytes32 depositRoot,
        uint256 nonce,
        bytes calldata depositCalldata,
        IDepositSecurityModule.Signature[] calldata sortedGuardianSignatures
    ) external onlyDelegateCall {
        if (IERC20(weth).balanceOf(address(this)) < amount)
            revert NotEnoughWeth();

        uint256 unfinalizedStETH = withdrawalQueue.unfinalizedStETH();
        uint256 bufferedEther = ISteth(steth).getBufferedEther();
        if (bufferedEther < unfinalizedStETH)
            revert InvalidWithdrawalQueueState();

        _wethToWSteth(amount);
        depositSecurityModule.depositBufferedEther(
            blockNumber,
            blockHash,
            depositRoot,
            stakingModuleId,
            nonce,
            depositCalldata,
            sortedGuardianSignatures
        );
    }
```

- So as can be seen, `StakingModule.convertAndDeposit()` will convert the transferred `wETH` to `wstETH` and deposited it in the `depositSecurityModule`, and the convert operation is done via `StakingModule._wethToWSteth()`:

```js
   function _wethToWSteth(uint256 amount) private {
        IWeth(weth).withdraw(amount);
        ISteth(steth).submit{value: amount}(address(0));
        IERC20(steth).safeIncreaseAllowance(address(wsteth), amount);
        IWSteth(wsteth).wrap(amount);
    }
```

## Impact

As can be noticed, the ` IWeth(weth).withdraw(amount)` will transfer the `ETH` to the staking `SimpleDVTStakingStrategy` contract, but since there's no `receive()` function implemented to receive the native token; the transaction will revert, rendering the `SimpleDVTStakingStrategy` unusable.


## Code Snippet

https://github.com/sherlock-audit/2024-06-mellow/blob/26aa0445ec405a4ad637bddeeedec4efe1eba8d2/mellow-lrt/src/strategies/SimpleDVTStakingStrategy.sol#L8C1-L11C2

## Tool used

Manual Review

## Recommendation

Update `SimpleDVTStakingStrategy` contract to have `receive()` function:

```diff
+   receive() external payable {
+       if (msg.sender != address(weth)) revert InvalidSender();
+   }
```