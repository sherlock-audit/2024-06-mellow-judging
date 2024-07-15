# Issue M-1: `ratiosX96Value` rounds in favor of user and not vault 

Source: https://github.com/sherlock-audit/2024-06-mellow-judging/issues/61 

## Found by 
Ironsidesec, X12, eeyore, hash
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



## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/mellow-finance/mellow-lrt/pull/44


# Issue M-2: Predefined amount parameter can challange allocation to Obol validators 

Source: https://github.com/sherlock-audit/2024-06-mellow-judging/issues/260 

## Found by 
hash
## Summary
Deposits need not be allocated to Obol validators

## Vulnerability Detail
In `StakingModule`, the amount to be deposited in Lido is supposed to go to the Obol validators. But the only check kept for this is that the `stakingModuleId` is set to `SimpleDVT` module id in Lido and `unfinalizedstETH <= bufferedETH`

[link](https://github.com/sherlock-audit/2024-06-mellow/blob/26aa0445ec405a4ad637bddeeedec4efe1eba8d2/mellow-lrt/src/modules/obol/StakingModule.sol#L48-L75)
```solidity
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

There is no enforcement/checks kept on the `amount` param. It is possible to pass in any value for the `amount` param or the buffered ETH amount in lido to change from the value using which the amount was calculated before the call due to other deposits . This allows for scenarios where the `amount` is not deposited into validators at all (not in multiples of 32 eth), could be shared with other staking modules or could result in reverts due to the max limit restriction in the lido side. There is also no enforcement that the deposits made would be allocated to Obol validators itself since the simpleDVT staking module in Lido also contains SSV operators

## Impact
The amount to be staked for obol validators can be assigned to other operators

## Code Snippet

## Tool used
Manual Review

## Recommendation
Consult with lido team to make it possible to deposit specifically to a certain set of operators and calculate the maximum amount depositable to that set at the instant of the call rather than a predefined value



## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/mellow-finance/mellow-lrt/pull/44


