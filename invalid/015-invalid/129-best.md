Cuddly Lava Sloth

Medium

# The Vault  Incompatibility with Fee on Transfer Tokens

## Summary

The vault is not compatible with fee on transfer tokens (e.g. [USDT, USDC](https://github.com/d-xo/weird-erc20?tab=readme-ov-file#fee-on-transfer)), which can lead to discrepancies in token amounts during deposits and withdrawals. This can result in inaccurate vault calculations because the actual value received and transferred to the users is not accounted for correctly.

## Vulnerability Detail

The issue arises in both the `Vault::deposit` , `Vault::processWithdrawals`  and  `Vault::emergencyWithdraw`  functions. When fee on transfer tokens are used, the actual amount of tokens transferred can be less than the specified amount due to the transfer fees. This discrepancy is not accounted for, leading to potential miscalculations in the vault.

`Vault.sol::deposit` 
```solidity 
for (uint256 i = 0; i < tokens.length; i++) {
            uint256 priceX96 = priceOracle.priceX96(address(this), tokens[i]);
            totalValue += totalAmounts[i] == 0
                ? 0
                : FullMath.mulDivRoundingUp(totalAmounts[i], priceX96, Q96);
            if (ratiosX96[i] == 0) continue;
            uint256 amount = FullMath.mulDiv(ratioX96, ratiosX96[i], Q96);
 @>           IERC20(tokens[i]).safeTransferFrom(
                msg.sender,
                address(this),
                amount
            );
            actualAmounts[i] = amount;
            depositValue += FullMath.mulDiv(amount, priceX96, Q96);
        }
```

`Vault::processWithdrawals`

```solidity
for (uint256 j = 0; j < s.tokens.length; j++) {
                s.erc20Balances[j] -= expectedAmounts[j];
@>                IERC20(s.tokens[j]).safeTransfer(
                    request.to,
                    expectedAmounts[j]
                );
            }
```
`Vault::emergencyWithdraw`
```solidity
for (uint256 i = 0; i < tokens.length; i++) {
            if (amounts[i] == 0) {
                if (minAmounts[i] != 0) revert InsufficientAmount();
                continue;
            }
            uint256 amount = FullMath.mulDiv(
                IERC20(tokens[i]).balanceOf(address(this)),
                request.lpAmount,
                totalSupply
            );
            if (amount < minAmounts[i]) revert InsufficientAmount();
@>            IERC20(tokens[i]).safeTransfer(request.to, amount);
            actualAmounts[i] = amount;
        }
```

## Impact
The vault loses money because it receives less than expected, and the `lpAmount` is inaccurately calculated, resulting in users receiving more than they should.

## Code Snippet
https://github.com/sherlock-audit/2024-06-mellow/blob/26aa0445ec405a4ad637bddeeedec4efe1eba8d2/mellow-lrt/src/Vault.sol#L329-L333
https://github.com/sherlock-audit/2024-06-mellow/blob/26aa0445ec405a4ad637bddeeedec4efe1eba8d2/mellow-lrt/src/Vault.sol#L562-L566
https://github.com/sherlock-audit/2024-06-mellow/blob/26aa0445ec405a4ad637bddeeedec4efe1eba8d2/mellow-lrt/src/Vault.sol#L409-L409

## Tool used

Manual Review

## Recommendation

Add functionality to accurately handle fee on transfer tokens. This can be done by comparing the token balances before and after the transfer to determine the actual amount received or transferred. 