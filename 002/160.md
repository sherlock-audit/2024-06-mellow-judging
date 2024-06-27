Square Mint Ape

High

# Incorrect TVL/Value calculations as token amounts are not normalized for tokens with decimals !=18

## Summary

The `priceX96` returned from the price oracle is intended to be used only with tokens that have 18 decimals. If USDC or USDT is used as underlying tokens, their amounts will not be normalized in TVL calculations, resulting in them being undervalued by a factor of 10^12.

## Vulnerability Detail

During the deposit or withdrawal process, the TVL of the vault and value of the deposit is calculated to determine the LP values to be minted or burned. If new tokens are added as underlying tokens that do not have 18 decimals, their values can be undervalued or overvalued by the system.

The lack of amount normalization can be seen in this example:

1. The vault is initially set to accept only WSTETH as the underlying token, and 40 WSTETH has been deposited so far.
2. A new underlying token, USDC, is added with the price oracle correctly set to the USDC/ETH price feed.
3. TVL is nominated in WETH, lets assume 1 WETH priced at approximately 3000 USDC in the market.
4. Alice deposits 1 WSTETH and 3000 USDC.
5. The correct price is returned from the USDC oracle but is designated for tokens with 18 decimals.
6. The 3000 USDC deposit is valued only as 0.000000000001 ETH.
7. Alice's LP tokens are minted at only about half of the real token value added.

## Impact

Incorrect TVL calculations can lead to user funds being lost or the vault being drained, especially when deposit and withdrawal token ratios are changed in a way that is unfavorable to the protocol.

## Code Snippet

https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L326
https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L335
https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L529

## Tool used

Manual Review

## Recommendation

Normalize amounts used in TVL/value calculations to 18 decimals (do not normalize `priceX96` itself as it is correctly used in `ratiosX96Value` calculations).

For example:
```diff
        for (uint256 i = 0; i < tokens.length; i++) {
            uint256 priceX96 = priceOracle.priceX96(address(this), tokens[i]);
            totalValue += totalAmounts[i] == 0
                ? 0
-                : FullMath.mulDivRoundingUp(totalAmounts[i], priceX96, Q96);
+                : FullMath.mulDivRoundingUp(totalAmounts[i] * 10**(18 - tokens[i].decimals()), priceX96, Q96);
            if (ratiosX96[i] == 0) continue;
            uint256 amount = FullMath.mulDiv(ratioX96, ratiosX96[i], Q96);
            IERC20(tokens[i]).safeTransferFrom(
                msg.sender,
                address(this),
                amount
            );
            actualAmounts[i] = amount;
-           depositValue += FullMath.mulDiv(amount, priceX96, Q96);
+           depositValue += FullMath.mulDiv(amount * 10**(18 - tokens[i].decimals()), priceX96, Q96);
        }
```
