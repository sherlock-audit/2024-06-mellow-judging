Uneven Mango Ferret

Medium

# Division by Zero in _processLpAmount Due to Equal Debt and Non-Debt Token Amounts leads to revert.

## Summary
The `_processLpAmount` function in the smart contract is used to mint new LP tokens based on the deposit value and the total value of underlying tokens. However, the calculation is vulnerable to a division by zero error if the total value of underlying tokens becomes zero, which can happen if the amounts of debt tokens and non-debt tokens are equal. This edge case can cause the function to revert, preventing the minting of LP tokens.
## Vulnerability Detail
The function _processLpAmount calculates the number of LP tokens to mint using the following formula:
```javascript 
lpAmount = FullMath.mulDiv(depositValue, totalSupply, totalValue);
```
Here, totalValue is the sum of all underlying tokens from the module. If the amounts of debt tokens and non-debt tokens are equal, totalValue will be zero. As a result, the formula will attempt to divide by zero, causing the function to revert. This edge case can occur if there are equal amounts of debt and non-debt tokens, leading to a total value of zero.
## Impact
An attacker can exploit this vulnerability to:
1.Cause the _processLpAmount function to revert due to a division by zero error.
2.Prevent the minting of LP tokens, disrupting the normal operation of the contract.
3.Create a Denial of Service (DoS) scenario by manipulating the amounts of debt and non-debt tokens to be equal, effectively stopping further deposits.
## Code Snippet
https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L346

```javascript 
function _processLpAmount(
        address to,
        uint256 depositValue,
        uint256 totalValue,
        uint256 minLpAmount
    ) private returns (uint256 lpAmount) {
        uint256 totalSupply = totalSupply();
        if (totalSupply == 0) {
            // scenario for initial deposit
            _requireAtLeastOperator();
            lpAmount = minLpAmount;
            if (lpAmount == 0) revert ValueZero();
            if (to != address(this)) revert Forbidden();
        } else {
            lpAmount = FullMath.mulDiv(depositValue, totalSupply, totalValue);
            if (lpAmount < minLpAmount) revert InsufficientLpAmount();
            if (to == address(0)) revert AddressZero();
        }

        if (lpAmount + totalSupply > configurator.maximalTotalSupply())
            revert LimitOverflow();
        _mint(to, lpAmount);
    }
```
## Tool used

Manual Review

## Recommendation
To mitigate this issue, add a check to ensure that totalValue is not zero before performing the division. If totalValue is zero, handle the scenario appropriately to prevent the division by zero error.
```diff
function _processLpAmount(
    address to,
    uint256 depositValue,
    uint256 totalValue,
    uint256 minLpAmount
) private returns (uint256 lpAmount) {
    uint256 totalSupply = totalSupply();
    if (totalSupply == 0) {
        // scenario for initial deposit
        _requireAtLeastOperator();
        lpAmount = minLpAmount;
        if (lpAmount == 0) revert ValueZero();
        if (to != address(this)) revert Forbidden();
    } else {
+        if (totalValue == 0) revert InvalidState(); // Add a check to prevent division by zero
        lpAmount = FullMath.mulDiv(depositValue, totalSupply, totalValue);
        if (lpAmount < minLpAmount) revert InsufficientLpAmount();
        if (to == address(0)) revert AddressZero();
    }

    if (lpAmount + totalSupply > configurator.maximalTotalSupply())
        revert LimitOverflow();
    _mint(to, lpAmount);
}
```