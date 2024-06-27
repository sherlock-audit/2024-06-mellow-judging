Bitter Jetblack Finch

Medium

# Deposit-withdraw arbitrage opportunities, allows an attacker to extract value from the protocol

## Summary

Users can arbitrage the protocol to make profit. More in detail, the protocol relies on Chainlink price feeds to fetch the prices for specific tokens relative to the base token of a particular vault,then are used in calculation with 96-bit expression. These prices are essentially used in deposit and withdraw functionalities.

```javascript
function getPrice(
        address vault,
        address token
    ) public view returns (uint256 answer, uint8 decimals) {
        return _validateAndGetPrice(_aggregatorsData[vault][token]);
    }

function priceX96(
        address vault,
        address token
    ) external view returns (uint256 priceX96_) {
        if (vault == address(0)) revert AddressZero();
        if (token == address(0)) revert AddressZero();
        address baseToken = baseTokens[vault];
        if (baseToken == address(0)) revert AddressZero();
        if (token == baseToken) return Q96;
        (uint256 tokenPrice, uint8 decimals) = getPrice(vault, token);
@>      (uint256 baseTokenPrice, uint8 baseDecimals) = getPrice(
            vault,
            baseToken
        );
        priceX96_ = FullMath.mulDiv(
            tokenPrice * 10 ** baseDecimals,
            Q96,
            baseTokenPrice * 10 ** decimals
        );
    }      
```

## Vulnerability Detail

However the price feeds of the different LST's have different deviation threshold and different heartbeats. To clarify, dev. threshold means that the price can be off by up to the given % compared to the real price. For example:

- [rETH/ETH](https://data.chain.link/feeds/ethereum/mainnet/reth-eth) has a heartbeat of 24 hours and a deviation threshold of 2%. 

This means, the nodes will not update an on-chain price, in case the boundaries are not reached within the 24h period. These deviations are significant enough to open arbitrage possibilities, which will impact the overall `rETH/ETH` exchange rate badly.

Here is a scenario:

1. A user notices that a LST token with a pending price change (~2%)
2. He deposits that LST token (the more expensive one) and gets minted the calculated LP amount, utilizing `priceX96()`, which is dependend on the price feeds: 

```javascript
function deposit(
        address to,
        uint256[] memory amounts,
        uint256 minLpAmount,
        uint256 deadline
    )
        external
        nonReentrant
        checkDeadline(deadline)
        returns (uint256[] memory actualAmounts, uint256 lpAmount)
    {
        ...
        IPriceOracle priceOracle = IPriceOracle(configurator.priceOracle());
        for (uint256 i = 0; i < tokens.length; i++) {
@>          uint256 priceX96 = priceOracle.priceX96(address(this), tokens[i]);
            totalValue += totalAmounts[i] == 0
                ? 0
                : FullMath.mulDivRoundingUp(totalAmounts[i], priceX96, Q96);
            if (ratiosX96[i] == 0) continue;
            uint256 amount = FullMath.mulDiv(ratioX96, ratiosX96[i], Q96);
            IERC20(tokens[i]).safeTransferFrom(
                msg.sender,
                address(this),
                amount
            );
            actualAmounts[i] = amount;
            depositValue += FullMath.mulDiv(amount, priceX96, Q96);
        }

        lpAmount = _processLpAmount(to, depositValue, totalValue, minLpAmount);

function _processLpAmount(
        address to,
        uint256 depositValue,
        uint256 totalValue,
        uint256 minLpAmount
    ) private returns (uint256 lpAmount) {
        uint256 totalSupply = totalSupply();
        if (totalSupply == 0) {
            // scenario for initial deposit
            _requireAtLeastOperator();
            lpAmount = minLpAmount;
            if (lpAmount == 0) revert ValueZero();
            if (to != address(this)) revert Forbidden();
        } else {
            lpAmount = FullMath.mulDiv(depositValue, totalSupply, totalValue);
            if (lpAmount < minLpAmount) revert InsufficientLpAmount();
            if (to == address(0)) revert AddressZero();
        }

        if (lpAmount + totalSupply > configurator.maximalTotalSupply())
            revert LimitOverflow();
        _mint(to, lpAmount);
    }        
``` 
3. Request a withdrawal for using another asset (the cheaper one)
4. Gets away with value more than initial deposit, thus would call it stealing value from users

The result will be guaranteed profit for the attacker, as he can even automate the process by creating a bot, which executes tx's only when there will be profit, thus taking 0% risks.

Similar type issues can be seen here:

- https://github.com/code-423n4/2024-04-renzo-findings/issues/426
- https://github.com/code-423n4/2023-11-kelp-findings/issues/584

## Impact

Since the result ends up, with an attacker leaving the protocol with more funds than deposited initially, thus stealing value from users/protocol, i would consider the: 

- Impact: High
- Likelihood: Low, as it requires a malicious actor + threshold deviations
- Overall: Medium

## Code Snippet

https://github.com/sherlock-audit/2024-06-mellow/blob/6defa2f3481ee3232ea256d0c1e4f9df9860ffa5/mellow-lrt/src/oracles/ChainlinkOracle.sol#L72-L97
https://github.com/sherlock-audit/2024-06-mellow/blob/6defa2f3481ee3232ea256d0c1e4f9df9860ffa5/mellow-lrt/src/Vault.sol#L285-L367


## Tool used

Manual Review

## Recommendation

Maybe the arbitraging can be avoided by underpricing the LST's on deposit and overpricing on withdrawals 