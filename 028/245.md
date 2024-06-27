Mythical Porcelain Swan

Medium

# Incorrect Oracle Data Handling in `setChainlinkOracles` Function

## Summary
The `setChainlinkOracles` function in the `ChainlinkOracle` contract has an inconsistency in handling zero address oracles (`aggregatorV3`). The function skips validation for these entries but assigns them in the subsequent loop, potentially storing invalid oracle data. This can lead to incorrect price calculations and unexpected contract behavior.

## Vulnerability Detail
The `setChainlinkOracles` function processes two arrays of equal length: `tokens` and `data_`. The first loop checks if `data_[i].aggregatorV3` is a zero address and skips validation for these entries using `continue`. However, the second loop assigns all entries from `data_` to the `_aggregatorsData` mapping, including those with zero addresses.
i.e 1. The first loop skips entries with zero addresses, effectively excluding them from validation.
2. The second loop assigns all entries, including those with zero addresses, to the `_aggregatorsData` mapping.

## Impact
Relying on invalid oracle data can lead to incorrect behavior in vault functionalities that depend on accurate price information.

## Code Snippet

https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/oracles/ChainlinkOracle.sol#L46

```solidity
function setChainlinkOracles(
    address vault,
    address[] calldata tokens,
    AggregatorData[] calldata data_
) external {
    IDefaultAccessControl(vault).requireAdmin(msg.sender);
    if (tokens.length != data_.length) revert InvalidLength();
    for (uint256 i = 0; i < tokens.length; i++) {
        if (data_[i].aggregatorV3 == address(0)) continue;
        _validateAndGetPrice(data_[i]);
    }
    for (uint256 i = 0; i < tokens.length; i++) {
        _aggregatorsData[vault][tokens[i]] = data_[i];
    }
    emit ChainlinkOracleSetChainlinkOracles(
        vault,
        tokens,
        data_,
        block.timestamp
    );
}
```

## Tool used

Manual Review

## Recommendation

To ensure only validated data is assigned, modify the second loop to skip entries with a zero address:

```solidity
function setChainlinkOracles(
    address vault,
    address[] calldata tokens,
    AggregatorData[] calldata data_
) external {
    IDefaultAccessControl(vault).requireAdmin(msg.sender);
    if (tokens.length != data_.length) revert InvalidLength();
    for (uint256 i = 0; i < tokens.length; i++) {
        if (data_[i].aggregatorV3 == address(0)) continue;
        _validateAndGetPrice(data_[i]);
    }
    for (uint256 i = 0; i < tokens.length; i++) {
        if (data_[i].aggregatorV3 == address(0)) continue; // Skip zero address entries
        _aggregatorsData[vault][tokens[i]] = data_[i];
    }
    emit ChainlinkOracleSetChainlinkOracles(
        vault,
        tokens,
        data_,
        block.timestamp
    );
}
```

This change ensures that only entries with valid `aggregatorV3` addresses are stored, maintaining the integrity of the oracle data.