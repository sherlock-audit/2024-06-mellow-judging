Tangy Chiffon Cricket

Medium

# Missing value check in `commitMaximalTotalSupply` function

## Summary

The vulnerability is attributable to the absence of a validation mechanism that ensures the newly staged maximal total supply is greater than or equal to the current total supply when it's time to commit this value, post the staging delay period.

## Vulnerability Detail

- The `stageMaximalTotalSupply` function correctly checks that the newly proposed `maximalTotalSupply_` cannot be less than the current total supply of the vault (`IVault(vault).totalSupply()`). If this condition is not met, the operation reverts.
- After staging, there is a delay (`_maximalTotalSupplyDelay`) before the new maximal total supply can be committed to ensure time for review and possibly, for objections to be raised.
- The issue here is that the total supply (`IVault(vault).totalSupply()`) could increase during the delay period, surpassing the staged maximal total supply to be committed when call `commitMaximalTotalSupply`.

## Impact

MaximalTotalSupply is wrongly set to a small value

## Code Snippet

https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/VaultConfigurator.sol#L200-L211

```solidity
    function stageMaximalTotalSupply(
        uint256 maximalTotalSupply_
    ) external onlyAdmin nonReentrant {
        if (maximalTotalSupply_ < IVault(vault).totalSupply())
            revert InvalidTotalSupply();
        _stage(_maximalTotalSupply, maximalTotalSupply_);
    }

    /// @inheritdoc IVaultConfigurator
    function commitMaximalTotalSupply() external onlyAdmin nonReentrant {
        _commit(_maximalTotalSupply, _maximalTotalSupplyDelay);
    }
```

## Tool used

Manual Review

## Recommendation

Add check in `commitMaximalTotalSupply` like `stageMaximalTotalSupply`