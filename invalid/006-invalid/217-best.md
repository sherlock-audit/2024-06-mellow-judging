Droll Ash Nuthatch

Medium

# User can benefit from `emergencyWithdrawal` in case other tokens are added

## Summary

`emergencyWithdrawal` doesn’t check for the request’s token hash and will transfer from all the currently active underlying tokens in case new ones are added, despite he deposited in a different set. 

## Vulnerability Detail

Emergency withdrawal can be used when requests cannot be satisfied in a normal way, **but due to missing checks whether the underlying tokens at the time of `emergencyWithdrawal` execution are the same as when the withdrawal request was created**. As a result, all the current underlying tokens + bond tokens received from deposits will be transferred to him, eventually leading to more profit than what he would have received from `processWithdrawal`.

```solidity
function emergencyWithdraw(
        uint256[] memory minAmounts,
        uint256 deadline
    )
        external
        nonReentrant
        checkDeadline(deadline)
        returns (uint256[] memory actualAmounts)
    {
        uint256 timestamp = block.timestamp;
        address sender = msg.sender;
        if (!_pendingWithdrawers.contains(sender)) revert InvalidState();
        WithdrawalRequest memory request = _withdrawalRequest[sender];
        if (timestamp > request.deadline) {
            _cancelWithdrawalRequest(sender);
            return actualAmounts;
        }

        if (
            request.timestamp + configurator.emergencyWithdrawalDelay() >
            timestamp
        ) revert InvalidState();

        uint256 totalSupply = totalSupply();
        (address[] memory tokens, uint256[] memory amounts) = baseTvl();
        if (minAmounts.length != tokens.length) revert InvalidLength();
        actualAmounts = new uint256[](tokens.length);
        for (uint256 i = 0; i < tokens.length; i++) {
            if (amounts[i] == 0) {
                if (minAmounts[i] != 0) revert InsufficientAmount();
                continue;
            }
            uint256 amount = FullMath.mulDiv(
                IERC20(tokens[i]).balanceOf(address(this)),
                request.lpAmount,
                totalSupply
            );
            if (amount < minAmounts[i]) revert InsufficientAmount();
            IERC20(tokens[i]).safeTransfer(request.to, amount);
            actualAmounts[i] = amount;
        }
        delete _withdrawalRequest[sender];
        _pendingWithdrawers.remove(sender);
        _burn(address(this), request.lpAmount);
        emit EmergencyWithdrawal(sender, request, actualAmounts);
    }
```

As we can see `baseTvl` is called that will take both all the underlying token balances in the vault itself + all the bonds lpTokens balances per underlying tokens. 

For example user has entered only in `wstETH` and later on decides to create withdrawal request:

```solidity
function registerWithdrawal(
      address to,
      uint256 lpAmount,
      uint256[] memory minAmounts,
      uint256 deadline,
      uint256 requestDeadline,
      bool closePrevious
  )
      external
      nonReentrant
      checkDeadline(deadline)
      checkDeadline(requestDeadline)
  {
      uint256 timestamp = block.timestamp;
      address sender = msg.sender;
      if (_pendingWithdrawers.contains(sender)) {
          if (!closePrevious) revert InvalidState();
          _cancelWithdrawalRequest(sender);
      }
      uint256 balance = balanceOf(sender);
      if (lpAmount > balance) lpAmount = balance;
      if (lpAmount == 0) revert ValueZero();
      if (to == address(0)) revert AddressZero();

      address[] memory tokens = _underlyingTokens;
      if (tokens.length != minAmounts.length) revert InvalidLength();

      WithdrawalRequest memory request = WithdrawalRequest({
          to: to,
          lpAmount: lpAmount,
          tokensHash: keccak256(abi.encode(tokens)),
          minAmounts: minAmounts,
          deadline: requestDeadline,
          timestamp: timestamp
      });
      _withdrawalRequest[sender] = request;
      _pendingWithdrawers.add(sender);
      _transfer(sender, address(this), lpAmount);
      emit WithdrawalRequested(sender, request);
  }
```

Knowing that there is a possibility more underlying tokens to be added he specifies `requestDeadline` above the `emergencyWithdrawalDelay` in order to make it possible to execute `emergencyWithdraw` in a latter stage. No normal withdrawals are being executed from `operators` because of either the low withdrawal requests or simply to maximise the yield from the underlying strategies by keeping the deposited funds inside bonds and **even adding another underlying tokens (rETH, weth, etc).** Then `emergencyWithdraw` will be the better option as it will give to the user from all of the circulating tokens in the system, then he can simply withdraw bond tokens in the Symbiotic and take the profit.

## Impact

EmergencyWithdrawal will transfer more tokens than normally processed withdraw in case there are underlying tokens added after request creation.

## Code Snippet
https://github.com/mellow-finance/mellow-lrt/blob/8191087682cc6a7f36c1c6390e37eb308953b11a/src/Vault.sol#L434-L473

https://github.com/mellow-finance/mellow-lrt/blob/8191087682cc6a7f36c1c6390e37eb308953b11a/src/Vault.sol#L371-L431

## Tool used

Manual Review

## Recommendation

Consider checking the underlying tokens hash in `vault::emergencyWithdraw` also.