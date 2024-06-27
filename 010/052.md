Creamy Malachite Condor

High

# Users can withdraw bellow `negativeAmounts` for a given token causing a complete DOS

## Summary
[_calculateTvl](https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L86) calculates the TVL by subtracting debt from the balance and will revert if the debt exceeds the balance. However, this check is not performed inside [emergencyWithdraw](https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L371), allowing users to withdraw below the debt limit. This can brick the entire contract as [deposit](https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L285), [emergencyWithdraw](https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L371), and [processWithdrawals](https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L536) will revert.

## Vulnerability Detail

**Where is the bug**

When [_calculateTvl](https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L86) calculates the vault's TVL, it checks [_tvls](https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L154) from every module. Some modules may return `data.isDebt` as `true`, making their returned amount a negative value.

```solidity
            for (uint256 j = 0; j < tokens.length; j++) {
                if (token != tokens[j]) continue;
                (data.isDebt ? negativeAmounts : amounts)[j] += amount;
                break;
            }
```

When the amount is negative, it is always subtracted from the positive one. If there are more `negativeAmounts` than `amounts` for a given token, the transaction will revert.

```solidity
        for (uint256 i = 0; i < tokens.length; i++) {
            if (amounts[i] < negativeAmounts[i]) revert InvalidState();
            amounts[i] -= negativeAmounts[i];
        }
```

**How the Bug Happens**

When users withdraw with [emergencyWithdraw](https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L371), they withdraw the bond shares directly from the vault. There is a check verifying if the returned amount is `amounts[i] == 0`, but nothing verifies if our withdraw amount is bigger than the positive vault amount (token amount - debt).

```solidity
        // amounts is not checked if it's > than our withdraw amount
        (address[] memory tokens, uint256[] memory amounts) = baseTvl();

        if (minAmounts.length != tokens.length) revert InvalidLength();
        actualAmounts = new uint256[](tokens.length);
        for (uint256 i = 0; i < tokens.length; i++) {
            if (amounts[i] == 0) {
                if (minAmounts[i] != 0) revert InsufficientAmount();
                continue;
            }
```

Users can withdraw more tokens than the difference between `negativeAmounts` and `amounts`, either maliciously or accidentally, causing [_calculateTvl](https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L86) to revert.

Example scenario:
- Debt for `tokenX` is really close to its amount, while other tokens have near 0 debt.
1. Withdraws happen as normal.
2. A user executes an [emergencyWithdraw](https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L371), as their withdraw has not been included in the previous batches.
3. The withdrawal processes as normal, but it withdraws enough of `tokenX` shares that now it's `negativeAmounts > amounts`
4. Another user tries to deposit some tokens, but the transaction reverts because `_calculateTvl` determines that there are more `negativeAmounts` than `amounts` of `tokenX`.

**Note:**
Currently, [ManagedTvlModule](https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/modules/erc20/ManagedTvlModule.sol) is the only one that can set custom debt. That debt can always be removed by an admin. However, an admin needs to intervene to prevent the vault from being completely bricked, which would mess up the internal accounting since the debt exists for a reason, creating a lose-lose situation.

## Impact
Vault DOS, user funds are stuck.

## Code Snippet
```solidity
            for (uint256 j = 0; j < tokens.length; j++) {
                if (token != tokens[j]) continue;
                (data.isDebt ? negativeAmounts : amounts)[j] += amount;
                break;
            }
```
## POC
Gist: https://gist.github.com/project-X12/cf3960455dbdd520fd0482d9ae1cf77d
Place inside: `mellow-lrt/tests/mainnet/unit/<name>.sol`
Run with: `forge test --fork-url <your url> --match-test test_POC_brick_vault -vvvv`

Result when we call `calculateStack`:

```jsx
    ├─ [26656] Vault::calculateStack() [staticcall]
      ... 
    │   ├─ [3678] ManagedTvlModule::tvl(Vault: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   └─ ← [Return] [Data({ token: 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2, underlyingToken: 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2, amount: 52250000004750000000 [5.225e19], underlyingAmount: 52250000004750000000 [5.225e19], isDebt: true })]
    │   └─ ← [Revert] InvalidState()
    └─ ← [Revert] InvalidState()
```

## Tool used
Manual Review

## Recommendation
Consider debt when processing withdrawals. The fix is simple - call [baseTvl](https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L122) inside [emergencyWithdraw](https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L371) and [underlyingTvl](https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L112) inside [processWithdrawals](https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L536). These functions will check if the debt is greater than the amounts and revert the withdrawal if so.