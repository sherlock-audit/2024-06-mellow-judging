Rapid Violet Scorpion

Medium

# RestrictingKeeper::processConfigurators() lacks call permission restrictions, anyone can perform revocation, and segmentation operations will not be performed as expected

## Summary
`RestrictingKeeper::processConfigurators()` lacks call permission restrictions, anyone can perform revocation, and segmentation operations will not be performed as expected
## Vulnerability Detail
```js
    // RestrictingKeeper::processConfigurators()
    function processConfigurators(
        VaultConfigurator[] memory configurators
    ) external {
        for (uint256 i = 0; i < configurators.length; i++) {
            VaultConfigurator configurator = configurators[i];
@>            configurator.rollbackStagedBaseDelay();
@>            configurator.rollbackStagedMaximalTotalSupplyDelay();
@>            configurator.rollbackStagedMaximalTotalSupply();
        }
    }
```
`RestrictingKeeper::processConfigurators()` will call the method in the `VaultConfigurator` contract that requires the `onlyAdmin` permission, although the contract `RestrictingKeeper` needs to obtain relevant authorization before executing the call. However, once `RestrictingKeeper` is authorized, since the `RestrictingKeeper::processConfigurators()` method itself lacks call restrictions, anyone can call this method to perform the corresponding operation.
```js
    // VaultConfigurator.sol
    /// @inheritdoc IVaultConfigurator
@>    function rollbackStagedBaseDelay() external onlyAdmin nonReentrant {
        _rollback(_baseDelay);
    }

    /// @inheritdoc IVaultConfigurator
    function rollbackStagedMaximalTotalSupplyDelay()
        external
@>        onlyAdmin
        nonReentrant
    {
        _rollback(_maximalTotalSupplyDelay);
    }
    /// @inheritdoc IVaultConfigurator
    function rollbackStagedMaximalTotalSupply()
        external
@>        onlyAdmin
        nonReentrant
    {
        _rollback(_maximalTotalSupply);
    }
```
### Poc
Please add to `tests/mainnet/unit/utils/RestrictingKeeperTest.t.sol` and run
```js
  function testEveryOneCanCallProcessConfigurators() external {
    // deploy RestrictingKeeper
    RestrictingKeeper keeper = new RestrictingKeeper();
    // admin grants role
    vm.startPrank(params.admin);
    setup.vault.grantRole(setup.vault.ADMIN_DELEGATE_ROLE(), address(keeper));
    vm.stopPrank();
    // init user
    address user = makeAddr("user");
    VaultConfigurator[] memory configurators = new VaultConfigurator[](1);
    configurators[0] = VaultConfigurator(address(setup.configurator));
    vm.prank(user);
    keeper.processConfigurators(configurators);
  }
  // Running 1 test for tests/mainnet/unit/utils/RestrictingKeeperTest.t.sol:Used
  // [PASS] testEveryOneCanCallProcessConfigurators() (gas: 332113)
  // Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.32s
```
## Impact
`RestrictingKeeper::processConfigurators()` lacks call permission restrictions, anyone can perform revocation, and segmentation operations will not be performed as expected
## Code Snippet
https://github.com/sherlock-audit/2024-06-mellow/blob/26aa0445ec405a4ad637bddeeedec4efe1eba8d2/mellow-lrt/src/utils/RestrictingKeeper.sol#L7-L16
https://github.com/sherlock-audit/2024-06-mellow/blob/26aa0445ec405a4ad637bddeeedec4efe1eba8d2/mellow-lrt/src/VaultConfigurator.sol#L213-L220
https://github.com/sherlock-audit/2024-06-mellow/blob/26aa0445ec405a4ad637bddeeedec4efe1eba8d2/mellow-lrt/src/VaultConfigurator.sol#L294-L297
https://github.com/sherlock-audit/2024-06-mellow/blob/26aa0445ec405a4ad637bddeeedec4efe1eba8d2/mellow-lrt/src/VaultConfigurator.sol#L483-L490

## Tool used

Manual Review

## Recommendation
Add corresponding caller permission control
