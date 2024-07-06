Creamy Malachite Condor

Medium

# `ratiosX96Value` rounds in favor of user and not vault

## Summary
`ratiosX96Value` is rounded down instead of up, causing withdrawals to favor users. This can slowly decrease the value in our vault, potentially leading to insolvency.

## Vulnerability Detail
`ratiosX96Value`, calculated in [calculateStack](https://github.com/mellow-finance/mellow-lrt/blob/dev-symbiotic-deploy/src/Vault.sol#L507),

```solidity
s.ratiosX96Value += FullMath.mulDiv(s.ratiosX96[i], priceX96, Q96);
```
 is used as a denominator inside [analyzeRequest](https://github.com/mellow-finance/mellow-lrt/blob/dev-symbiotic-deploy/src/Vault.sol#L476) to calculate `coefficientX96` and the user's `expectedAmounts`.

```solidity
uint256 value = FullMath.mulDiv(lpAmount, s.totalValue, s.totalSupply);
value = FullMath.mulDiv(value, D9 - s.feeD9, D9);
uint256 coefficientX96 = FullMath.mulDiv(value, Q96, s.ratiosX96Value);
...
expectedAmounts[i] = ratiosX96 == 0 ? 0 : FullMath.mulDiv(coefficientX96, ratiosX96, Q96);
```

However, [calculateStack](https://github.com/mellow-finance/mellow-lrt/blob/dev-symbiotic-deploy/src/Vault.sol#L507) rounds the denominator down (thanks to `mulDiv`) , which increases `coefficientX96` and thus increases what users withdraw.

This is unwanted behavior in vaults. Rounding towards users decreases the vault's value and can ultimately cause insolvency. The previous audit found a similar issue in the deposit function - [M1](https://github.com/mellow-finance/mellow-lrt/blob/dev-symbiotic-deploy/audits/202406_Statemind/Mellow%20LRT%20report%20with%20deployment.pdf).

## Impact
The vault may become insolvent or lose small amounts of funds with each withdrawal.

## Code Snippet
```solidity
for (uint256 i = 0; i < tokens.length; i++) {
    uint256 priceX96 = priceOracle.priceX96(address(this), tokens[i]);
    s.totalValue += FullMath.mulDiv(amounts[i], priceX96, Q96);
    s.ratiosX96Value += FullMath.mulDiv(s.ratiosX96[i], priceX96, Q96);
    s.erc20Balances[i] = IERC20(tokens[i]).balanceOf(address(this));
}
```

## Tool used
Manual Review

## Recommendation
Round the value up instead of down, similar to how it's done inside [deposit](https://github.com/mellow-finance/mellow-lrt/blob/dev-symbiotic-deploy/src/Vault.sol#L326). 

```diff
for (uint256 i = 0; i < tokens.length; i++) {
    uint256 priceX96 = priceOracle.priceX96(address(this), tokens[i]);
    s.totalValue += FullMath.mulDiv(amounts[i], priceX96, Q96);
-   s.ratiosX96Value += FullMath.mulDiv(s.ratiosX96[i], priceX96, Q96);
+   s.ratiosX96Value += FullMath.mulDivRoundingUp(s.ratiosX96[i], priceX96, Q96);
    s.erc20Balances[i] = IERC20(tokens[i]).balanceOf(address(this));
}
```