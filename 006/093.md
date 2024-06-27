Curved Tiger Peacock

High

# Lack of checking on return values when calling external functions

## Summary
Many functions in the protocol call external functions using `call` and `delegateCall`, but they do not check the return value. If the external execution fails and the protocol does not revert, it can result in a loss of funds.

## Vulnerability Detail
In the `vault.externalCall()` and `delegateCall()` functions, as well as in the `SimpleDVTStakingStrategy.convertAndDeposit()` function, the protocol calls external contracts using methods like `to.call(data)` and `to.delegatecall(data)`. 

```solidity
    function externalCall(
        address to,
        bytes calldata data
    ) external nonReentrant returns (bool success, bytes memory response) {
        _requireAtLeastOperator();
        if (configurator.isDelegateModuleApproved(to)) revert Forbidden();
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


(success, response) = to.delegatecall(data);

 (success, ) = vault.delegateCall(
            address(stakingModule),
            abi.encodeWithSelector(
                IStakingModule.convertAndDeposit.selector,
                amount,
                blockNumber,
                blockHash,
                depositRoot,
                nonce,
                depositCalldata,
                sortedGuardianSignatures
            )
        );
        emit ConvertAndDeposit(success, msg.sender);

   vault.delegateCall(
            address(stakingModule),
            abi.encodeWithSelector(
                IStakingModule.convert.selector,
                amountForStake
            )
        );

```

However, the protocol does not check the returned `success` value from these calls. If the external execution fails but the protocol has already transferred funds to the external contract without reverting the transaction, it could result in a loss of funds.

## Impact
It could result in a loss of funds.
## Code Snippet
https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L267-L282
https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L249-L264
https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/strategies/SimpleDVTStakingStrategy.sol#L45
https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/strategies/SimpleDVTStakingStrategy.sol#L72

## Tool used

Manual Review

## Recommendation
Check the returned success value.
