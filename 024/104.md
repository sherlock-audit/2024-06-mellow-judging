Breezy Syrup Cricket

Medium

# Lack of Slippage Protection in Deposits

## Summary
The `deposit` function lacks robust slippage protection, which could lead to users receiving fewer LP tokens than expected in volatile market conditions.


## Vulnerability Detail
While the function does have a `minLpAmount` parameter, it doesn't account for potential price changes between the time a user submits a transaction and when it's mined. This could result in fewer LP tokens being minted than the user expects.


## Impact
Users could receive fewer LP tokens than anticipated, especially in volatile market conditions, leading to potential financial losses.
https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L287-L346

## Code Snippet
```solidity
function deposit(
    address to,
    uint256[] memory amounts,
    uint256 minLpAmount,
    uint256 deadline
) external nonReentrant checkDeadline(deadline) returns (uint256[] memory actualAmounts, uint256 lpAmount) {
    // ... (code omitted for brevity)
    lpAmount = _processLpAmount(to, depositValue, totalValue, minLpAmount);
    // ... (code omitted for brevity)
}
```

## Tool used

Manual Review

## Recommendation
Implement more robust slippage protection by allowing users to specify a maximum amount of each token they're willing to deposit for a given amount of LP tokens. This could be done by adding a maxAmounts parameter to the deposit function.
