Ancient Pastel Beaver

High

# `Vault::emergencyWithdraw()` allows users to avoid paying withdraw fee

## Summary

`Vault::emergencyWithdraw()` does not apply any kind of fee on withdrawing unlike how the system is shown to behave through `Vault::processWithdrawals` where there is a fee applied.


## Vulnerability Detail

As per the protocol overview: 

"In a regular withdrawal, the user registers a withdrawal request, after which the operator must perform a series of operations to ensure there are enough underlyingTokens on the vault's balance to fulfill the user's request. Subsequently, the operator must call the processWithdrawals function.

If a user's request is not processed within the emergencyWithdrawalDelay period, the user can perform an emergency withdrawal. Note! In this case, the user may receive less funds than entitled by the system, as this function only handles ERC20 tokens in the system. Therefore, if the system has a base asset that is not represented as an ERC20 token, the corresponding portion of the funds will be lost by the user."

`Vault::emergencyWithdraw` allows users to withdraw their tokens but be at risk of loosing out on some of the funds they were entitled to.  This can most likely be avoided by frontrunning `DefaultBondStrategy::depositCallback` being called by the operator and withdrawing from the tokens available because calling emergency withdraw removes the ratio of funds requirement as there is no check inside the function. 

When `Vault::processWithdrawals` is called the function then calls `Vault::calculateStack` where a withdraw fee is applied.

```solidity
    /// @inheritdoc IVault
    function calculateStack()
        public
        view
        returns (ProcessWithdrawalsStack memory s)
    {
        (address[] memory tokens, uint256[] memory amounts) = underlyingTvl();
        s = ProcessWithdrawalsStack({
            tokens: tokens,
            ratiosX96: IRatiosOracle(configurator.ratiosOracle())
                .getTargetRatiosX96(address(this), false),
            erc20Balances: new uint256[](tokens.length),
            totalSupply: totalSupply(),
            totalValue: 0,
            ratiosX96Value: 0,
            timestamp: block.timestamp,
@>          feeD9: configurator.withdrawalFeeD9(),
            tokensHash: keccak256(abi.encode(tokens))
        });

        IPriceOracle priceOracle = IPriceOracle(configurator.priceOracle());
        for (uint256 i = 0; i < tokens.length; i++) {
            uint256 priceX96 = priceOracle.priceX96(address(this), tokens[i]);
            s.totalValue += FullMath.mulDiv(amounts[i], priceX96, Q96);
            s.ratiosX96Value += FullMath.mulDiv(s.ratiosX96[i], priceX96, Q96);
            s.erc20Balances[i] = IERC20(tokens[i]).balanceOf(address(this));
        }
    }
```
The fee is afterwards deducted inside `Vault::analyzeRequest`

```solidity
    /// @inheritdoc IVault
    function analyzeRequest(
        ProcessWithdrawalsStack memory s,
        WithdrawalRequest memory request
    ) public pure returns (bool, bool, uint256[] memory expectedAmounts) {
        uint256 lpAmount = request.lpAmount;
        if (
            request.tokensHash != s.tokensHash || request.deadline < s.timestamp
        ) return (false, false, expectedAmounts);

        uint256 value = FullMath.mulDiv(lpAmount, s.totalValue, s.totalSupply);
@>      value = FullMath.mulDiv(value, D9 - s.feeD9, D9);
        uint256 coefficientX96 = FullMath.mulDiv(value, Q96, s.ratiosX96Value);

        uint256 length = s.erc20Balances.length;
        expectedAmounts = new uint256[](length);
        for (uint256 i = 0; i < length; i++) {
            uint256 ratiosX96 = s.ratiosX96[i];
            expectedAmounts[i] = ratiosX96 == 0
                ? 0
                : FullMath.mulDiv(coefficientX96, ratiosX96, Q96);
            if (expectedAmounts[i] >= request.minAmounts[i]) continue;
            return (false, false, expectedAmounts);
        }
        for (uint256 i = 0; i < length; i++) {
            if (s.erc20Balances[i] >= expectedAmounts[i]) continue;
            return (true, false, expectedAmounts);
        }
        return (true, true, expectedAmounts);
    }
```

That essentially means that its more beneficial for users to use emergency withdraw in order to avoid fees and even make choices on what kind of tokens to receive without the ratio limiter.

## Impact

The main problem is that the contract would be losing out on withdraw fees which it was supposed to get.

The secondary issues is that since users will be using `Vault::emergencyWithdraw` by frontrunning the depositing of funds into bonds the contract will lose out on profits from bonds as the deposits will be smaller.


## Code Snippet

https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L371-L416

https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L476-L533

https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L536-L582

## Tool used

Manual Review

## Recommendation

Make sure the fee is applied inside `Vault::emergencyWithdraw`.