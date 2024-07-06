Steep Misty Raven

High

# Function `externalCall` from `src/Vault.sol` allows code from unapproved modules to be executed and forbids code from approved modules to be executed

## Summary
Function `externalCall` from `src/Vault.sol` performs a check for approved modules, which is incorrectly impelmented. As a result, approved modules can't be called and unapproved modules can be called.

## Vulnerability Detail
The incorrectly implemented check in the `externalCall` function from `src/Vault.sol`:

https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L254

This check is incorrectly implemented. Thus, approved modules can't be called, while unapproved modules can be called.

## Impact
Impact is severe as the check:

https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L254

is logically inverted - this allows code from unapproved modules to be executed, while code from already approved modules can't be executed.

## Code Snippet
https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L254

## Tool used
Manual Review

## Recommendation
The check should be inverted in order to be correct. Amend `externalCall` from `src/Vault.sol` in the following way:

```diff
    function externalCall(
        address to,
        bytes calldata data
    ) external nonReentrant returns (bool success, bytes memory response) {
        _requireAtLeastOperator();
-       if (configurator.isDelegateModuleApproved(to)) revert Forbidden();
+       if (!configurator.isDelegateModuleApproved(to)) revert Forbidden();
        IValidator validator = IValidator(configurator.validator());
        validator.validate(
            msg.sender,
            address(this),
            abi.encodeWithSelector(msg.sig, to, data)
        );
        validator.validate(address(this), to, data);
        (success, response) = to.call(data);
        emit ExternalCall(to, data, success, response);
    }
```
