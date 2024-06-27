Energetic Slate Panther

Medium

# transfer of vault tokens will still work when transfers are paused

## Summary
The `Vault::_update(...)` function is used extensively during vault lp token transfers. The function is supposed to revert when LP token transfers are currently locked. However due to the implementation of the function, transfers will still work when `configurator.areTransfersLocked() == true`.

```solidity
File: Vault.sol
586:     function _update(
587:         address from,
588:         address to,
589:         uint256 value
590:     ) internal override {
591:         if (configurator.areTransfersLocked()) {
592:             address this_ = address(this);
593:             address zero_ = address(0);
594: @>          if (from != this_ && to != this_ && from != zero_ && to != zero_)
595:                 revert Forbidden();
596:         }
597: 
598:         super._update(from, to, value);
599:     }

```

below is a breakdown of the above conditions as shown, the condition expects the sender and recipients to fulfill two states at once

```solidity
    from != this_ => not transfering form vault
    to != this_ => not withdrawing underlying
    from != zero_ => not minting
    to != zero_ => not burning
```
The problem is that the implemented condition will not cause the function to revert and as such locking transfers will not work as intended breaking core protocol functionality

## Vulnerability Detail
For instance,
- if `from == Alice` and `to == bob` it will not revert because the condition will evaluate to false


## Impact
This breaks core protocol functionality

## Code Snippet
https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L594-L595

## Tool used

Manual Review

## Recommendation
Modify the `_update(...)` function as shown below
```solidity
File: Vault.sol
586:     function _update(
587:         address from,
588:         address to,
589:         uint256 value
590:     ) internal override {
591:         if (configurator.areTransfersLocked()) {
592:             address this_ = address(this);
593:             address zero_ = address(0);

594:  -          if (from != this_ && to != this_ && from != zero_ && to != zero_)
594:  +         if (from != this_ || to != this_ || from != zero_ || to != zero_)

595:                 revert Forbidden();
596:         }
597: 
598:         super._update(from, to, value);
599:     }

```