Creamy Malachite Condor

Medium

# WStethRatiosAggregatorV3 will not work properly if the base token is anything but stETH

## Summary
Vaults have impermanent loss because they rely on the current price.

## Vulnerability Detail
The initial base token for our system is WETH, as seen [here](https://mellowprotocol.notion.site/Contracts-deployment-4cd6b91d9aef416291eb510d898f3841).

> baseTokens - Returns the base token associated with a specific vault. - baseTokens[Vault Address] = Wrapped ETH Address

With that in mind, when we extract the price from [WStethRatiosAggregatorV3](), our [latestRoundData](https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/oracles/WStethRatiosAggregatorV3.sol#L19) will return `IWSteth(wsteth).getStETHByWstETH`, which is the price of `wstETH` in `stETH` terms based on [Lido's docs](https://docs.lido.fi/contracts/wsteth/#getstethbywsteth). 

However, `WETH : stETH` can have small price differences (i.e., it's not `1 : 1`), and the ratio of `wstETH` in `stETH` will result in slight price differences when used with our base token - `WETH`. This will cause the real price of `wstETH` to be over or undervalued, resulting in users depositing fewer or more shares than they should.

Note that even if the base token is something else (other than `stETH`), the same issue will appear.

## Impact
Slight price differences can cause inaccurate share allocations.

## Code Snippet
```solidity
function getAnswer() public view returns (int256) {
    return int256(IWSteth(wsteth).getStETHByWstETH(10 ** decimals));
}

function latestRoundData() public view override returns (uint80, int256, uint256, uint256, uint80) {
    return (0, getAnswer(), block.timestamp, block.timestamp, 0);
}
```

## Tool used
Manual Review

## Recommendation
Have [WStethRatiosAggregatorV3](https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/oracles/WStethRatiosAggregatorV3.sol) perform another conversion for `stETH` to the base token and use that price.