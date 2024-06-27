Silly Lava Manatee

Medium

# DepositWrapper could become non-usable

## Summary
The devs declared that there would be more that 1 underlying tokens in the protocol. Additionally, the Vault.sol has a method to add underlying token, so it is clear that the underlying token.length could be more than 1.

```
As underlyingTokens only [weth, wsteth, reth, usdc, usdt] can be used
actually underlying tokens are only [wsteth, weth] + new ones
```

However, it creates the problem for the user who would like to deposit via the DepositWrapper. Let’s see how exactly. 

In the DepositWrapper in the deposit function there is a check which revert the function if the underlyingTokens.length > 1. It means that once the vault.underlyingTokens() length would become > 1, the deposit function would become non-usable.

```solidity
 address[] memory tokens = vault.underlyingTokens();
        if (tokens.length != 1 || tokens[0] != wsteth)
            revert InvalidTokenList();
```

## Vulnerability Detail
- Owner add one more underlying token (apart from the wsETH)
- User tries to deposit stETH via the DepositWrapper
- Since `tokens.length != 1`  but it is equal to 2 the function would revert.

## Impact
DepositWrapper contract becomes non usable once the underlying token length > 1

## Code Snippet
https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/utils/DepositWrapper.sol#L52-L53

## Tool used
Manual Review

## Recommendation
```solidity
if (tokens[0] != wsteth)
 revert InvalidTokenList();
```
