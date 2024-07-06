Suave Felt Fly

High

# [H-2] - DoS attack to prevent the `processWithdrawals` call from the `SimpleDVTStakingStrategy` contract.

## Summary
The `SimpleDVTStakingStrategy::processWithdrawals` function will attempt to process the withdrawals of eligible users from the `Vault` contract. It will then convert the `wETH` token to `wstETH` token by calling the `StakingModule::convert` function, and in the end it will check the remaining `wstETH` balance of the `Vault` contract to see if the remaining `balance > maxAllowedRemainder`.

If the `balance > maxAllowedRemainder` then it will `revert LimitOverflow()`.

## Vulnerability Detail
A malicious user can see the `operator` calling the `SimpleDVTStakingStrategy::processWithdrawals` function and they can frontrun that transaction in order to transfer in an amount of `wstETH` tokens that will make the remaining `balance > maxAllowedRemainder` and this will make the transaction revert.

## Impact

Although the malicious user has no financial incentives to behave like this, they can still cause a potential DoS attack that will prevent the `operator` from [processing withdrawals](https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/strategies/SimpleDVTStakingStrategy.sol#L83).

## Code Snippet

```solidity
    function processWithdrawals(...) external returns (bool[] memory statuses) {
//..
//..
        address wsteth = stakingModule.wsteth();
        uint256 balance = IERC20(wsteth).balanceOf(address(vault));
 //@audit | DoS by sending wsteth directly into the contract to increase its balance
        if (balance > maxAllowedRemainder) revert LimitOverflow();
    }
```

## Tool used

Manual Review

## Recommendation

Consider doing a "sweep" if the remaining `balance` is bigger than `maxAllowedRemainder` to prevent this from happening. This may not be the best fix, but it's an idea.

```diff
    function processWithdrawals(...) external returns (bool[] memory statuses) {
//..
//..
        address wsteth = stakingModule.wsteth();
        uint256 balance = IERC20(wsteth).balanceOf(address(vault));
+       if (balance > maxAllowedRemainder) {
+           uint256 remainder = balance - maxAllowedRemainder;
+           safeTransfer(address(vault), //some safe address, remainder );

}
    }
```
