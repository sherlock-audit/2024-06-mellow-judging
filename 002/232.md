Uneven Mango Ferret

High

# Inconsistent LP Token Minting Due to Dynamic TVL Module Addition and Removal

## Summary
The _processLpAmount function in the contract calculates the minting rate of LP tokens based on the total value locked (TVL) of underlying assets. However, an admin can add or remove TVL modules at any time, causing significant changes to the totalValue and leading to inconsistent minting rates. This allows users to exploit the system by front-running these changes, resulting in unfair distribution of minted tokens for equivalent deposits.
## Vulnerability Detail
The minting process for LP tokens is calculated using the formula:
```javascript 
lpAmount = FullMath.mulDiv(depositValue, totalSupply, totalValue);

```
Here, totalValue represents the sum of all underlying tokens across modules. If the admin removes a TVL module, totalValue drops, which increases the number of LP tokens minted for subsequent deposits. Conversely, if a new module with significant TVL is added, totalValue increases, reducing the number of LP tokens minted for subsequent deposits. This creates an imbalance, as users who deposit before the addition of a new module receive more LP tokens compared to those who deposit afterward, even if the deposit amounts are identical.
## Impact
Inconsistent Minting Rates: Users can receive significantly different amounts of LP tokens for the same deposit value depending on the state of TVL modules.
Imbalanced Withdrawals: Users who deposited before a TVL increase have more LP tokens, causing imbalances when they withdraw compared to new users who deposited the same amount.
## Code Snippet
https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L346
https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L227

```javascript 
    function addTvlModule(address module) external nonReentrant {
        _requireAdmin();
        ITvlModule.Data[] memory data = ITvlModule(module).tvl(address(this));
        for (uint256 i = 0; i < data.length; i++) {
            if (!_isUnderlyingToken[data[i].underlyingToken])
                revert InvalidToken();
        }
        if (!_tvlModules.add(module)) {
            revert AlreadyAdded();
        }
        emit TvlModuleAdded(module);
    }

    /// @inheritdoc IVault
    function removeTvlModule(address module) external nonReentrant {
        _requireAdmin();
        if (!_tvlModules.contains(module)) revert InvalidState();
        _tvlModules.remove(module);
        emit TvlModuleRemoved(module);
    }
```
## Tool used

Manual Review

## Recommendation
Implement a Timelock Mechanism: Introduce a delay for adding or removing TVL modules to prevent sudden changes in totalValue.