Clumsy Clay Mole

Medium

# `ChainlinkOracle._validateAndGetPrice()` doesn't correctly validate the returned price

## Summary

`ChainlinkOracle._validateAndGetPrice()` doesn't correctly validate the returned price as it allows the returned price to be zero or out of an acceptable price range (irreasonable price validation).

## Vulnerability Detail

- `ChainlinkOracle._validateAndGetPrice()` function is invoked whenever `getPrice()` is invoked to fetch the latest price for a specific token from the given vault's associated Chainlink oracle:

```js

    function _validateAndGetPrice(
        AggregatorData memory data
    ) private view returns (uint256 answer, uint8 decimals) {

        if (data.aggregatorV3 == address(0)) revert AddressZero();

        (, int256 signedAnswer, , uint256 lastTimestamp, ) = IAggregatorV3(
            data.aggregatorV3
        ).latestRoundData();
        // The roundId and latestRound are not used in the validation process to ensure compatibility
        // with various custom aggregator implementations that may handle these parameters differently
        if (signedAnswer < 0) revert InvalidOracleData();
        answer = uint256(signedAnswer);
        if (block.timestamp - data.maxAge > lastTimestamp) revert StaleOracle();
        decimals = IAggregatorV3(data.aggregatorV3).decimals();
    }
```

where the fetched priced is checked for staleness (`block.timestamp - data.maxAge > lastTimestamp`) and if its's `signedAnswer < 0`.

- But this doesn't correctly validate that the fetched price:

1. The `IAggregatorV3(data.aggregatorV3).latestRoundData()` could return a zero price if no answer has been reached.
2. The returned price could be out of an acceptable range, as there's no `minAnswer/maxAnswer` circuit breaker checks on the returned price, and this could be as a result of a huge drop/pump of the asset value where the oracle will continue returning the minAnswer/maxAnswer instead of the actual market price of the asset.

## Impact

This would result in using incorrect assets prices fetched by the `ChainlinkOracle.priceX96()`, where this function is invoked by the `Vault` to evaluate the total value locked, thus resulting in incorrect shares being minted to the users (lp tokens) and incorrect values redeemed (withdrawn) by the users.

## Code Snippet

https://github.com/sherlock-audit/2024-06-mellow/blob/26aa0445ec405a4ad637bddeeedec4efe1eba8d2/mellow-lrt/src/oracles/ChainlinkOracle.sol#L56C5-L69C6

## Tool used

Manual Review

## Recommendation

Update `ChainlinkOracle._validateAndGetPrice()` to check against zero price if returned, and to validate the fetched price is within a an acceptable range (add `minPrice` & `maxPrice` to the `_aggregatorsData[vault][token]`).