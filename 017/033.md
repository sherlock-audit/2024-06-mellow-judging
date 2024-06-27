Creamy Malachite Condor

Medium

# Protocol cannot receive any referrals

## Summary
[DepositWrapper](https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/utils/DepositWrapper.sol#L31)'s `submit` can be used by referees to generate referral revenue. Setting it to `address(0)` removes this ability permanently.

## Vulnerability Detail
Lido's `submit` function can generate revenue from referrals ( [docs](https://docs.lido.fi/contracts/lido#submit-1) ) . In the current implementation, the referral address is hard-coded to `address(0)`, preventing the system from ever receiving revenue from referrals.

For example, if the DAO is approved for Lido's referral program in the future, it will not be able to place itself as the referrer because the value is hard-coded to `address(0)`.

## Impact
The protocol won't be able to receive referral revenue.

## Code Snippet
```solidity
    function _ethToWsteth(uint256 amount) private returns (uint256) {
        ISteth(steth).submit{value: amount}(address(0));
        return _stethToWsteth(amount);
    }
```

## Tool used
Manual Review

## Recommendation
Instead of `address(0)`, set it to an address variable so it can be changed in the future if needed.
```diff
    function _ethToWsteth(uint256 amount) private returns (uint256) {
-       ISteth(steth).submit{value: amount}(address(0));
+       ISteth(steth).submit{value: amount}(refAddress);
        return _stethToWsteth(amount);
    }
```