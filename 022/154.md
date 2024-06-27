Tangy Chiffon Cricket

High

# Proposal will be disabled forever if it is skipped by `acceptProposal`

## Summary

Proposal will be disabled forever if it is skipped by `acceptProposal`

## Vulnerability Detail

1. if there are 4 proposal  in contract, and the `latestAcceptedNonce` is 2
2. it means that prop#3 and prop#4 are pending for accepting.
3. if we call `acceptProposal(4)`, it will success and set `latestAcceptedNonce` to 4
4. the prop#3 will be disabled forever because `index`(3) is less than `latestAcceptedNonce` (4)

```solidity
        if (index <= latestAcceptedNonce || _proposals.length < index)
            revert Forbidden();
```


## Impact

the proposal will be disabled forever

## Code Snippet

https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/security/AdminProxy.sol#L135-L146

```solidity
    /// @inheritdoc IAdminProxy
    function acceptProposal(uint256 index) external onlyAcceptor {
        if (index <= latestAcceptedNonce || _proposals.length < index)
            revert Forbidden();
        Proposal memory proposal = _proposals[index - 1];
        proxyAdmin.upgradeAndCall(
            proxy,
            proposal.implementation,
            proposal.callData
        );
        latestAcceptedNonce = index;
        emit ProposalAccepted(index, tx.origin);
    }
```

## Tool used

Manual Review

## Recommendation

Maybe add a new function to move old proposal to the last of '_proposals'.
