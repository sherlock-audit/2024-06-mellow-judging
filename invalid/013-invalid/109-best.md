Energetic Slate Panther

Medium

# Users can DOS withdrawal processing if they get blacklisted after

## Summary
Per the changes made duriing the course of the audit on discord announced by @wang [here](https://discord.com/channels/812037309376495636/1252276613417013353/1254895536629219428) and confirmed by @armoking32 [here](https://discord.com/channels/812037309376495636/1252276613417013353/1254895536629219428) that the protocols whitelisted tokens now include USDT and USDC.


If a user specifies an address that has been blacklisted by USDC as the recipient of a withdrawal request, the `processWithdrawals(...)` function will revert.



## Vulnerability Detail

`processWithdrawals(...)` is used to process withdrawals for an array of users and the expected amount of each user who has requested a withdrawal will be sent the address (`request.to`) specified by the user who made the request. However, if the specified recipient has been blacklisted by USDC, the `processWithdrawals(...)` function will revert blocking withdrawals for other users

```solidity
File: Vault.sol
538:     /// @inheritdoc IVault
539:     function processWithdrawals(
540:         address[] memory users
541:     ) external nonReentrant returns (bool[] memory statuses) {
SNIP
......
562: 
563:             for (uint256 j = 0; j < s.tokens.length; j++) {
564:                 s.erc20Balances[j] -= expectedAmounts[j];
565:                 IERC20(s.tokens[j]).safeTransfer(
566:                     request.to,
567:    @>               expectedAmounts[j]
568:                 );
569:             }
SNIP
.......
584:         emit WithdrawCallback(callback);
585:     }

```

## Impact
This can lead to a Denial of Service for users when redeeming their lp token for their underlying asset

## Code Snippet
https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L461-L468

## Tool used
Manual Review

## Recommendation
Modify the `processWithdrawals(...)` function to use a `try/catch` as shown below

```solidity

File: Vault.sol

538:     /// @inheritdoc IVault
539:     function processWithdrawals(
540:         address[] memory users
541:     ) external nonReentrant returns (bool[] memory statuses) {
SNIP
......
562: 

563:  -           for (uint256 j = 0; j < s.tokens.length; j++) {
564:  -               s.erc20Balances[j] -= expectedAmounts[j];
565:  -               IERC20(s.tokens[j]).safeTransfer(
566:  -                   request.to,
567:  -                   expectedAmounts[j]
568:  -               );
569:  -           }

563:  +           for (uint256 j = 0; j < s.tokens.length; j++) {
564:  +               s.erc20Balances[j] -= expectedAmounts[j];
565:  +               try IERC20(s.tokens[j]).safeTransfer(
566:  +                   request.to,
567:  +                   expectedAmounts[j]
568:  +               ) {} catch { continue }
569:  +           }

SNIP
.......
584:         emit WithdrawCallback(callback);
585:     }
```