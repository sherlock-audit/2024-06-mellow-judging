Droll Ash Nuthatch

High

# Vault::removeToken can be griefed

## Summary

Sending a single `wei` of the underlying token that is to be removed will block the removal permanently because there is no way the excess tokens to be transferred out of the vault.

## Vulnerability Detail

In order for an underlying token to be removed from a Vault his `underlying` balances (**both idle in the Vault and deposited into the underlying strategies**) should be zero as can be seen in `Vault::removeToken`:

```solidity
function removeToken(address token) external nonReentrant {
    _requireAdmin();
    if (!_isUnderlyingToken[token]) revert InvalidToken();
    (address[] memory tokens, uint256[] memory amounts) = underlyingTvl();
    uint256 index = tokens.length;
    for (uint256 i = 0; i < tokens.length; i++) {
        if (tokens[i] == token) {
            if (amounts[i] != 0) revert NonZeroValue();//@audit can be griefed
            index = i;
            break;
        }
    }
    _isUnderlyingToken[token] = false;
    while (index + 1 < tokens.length) {
        _underlyingTokens[index] = tokens[index + 1];
        index++;
    }
    _underlyingTokens.pop();
    emit TokenRemoved(token);
}
```

The problem is that everyone can transfer 1 wei directly to the Vault, resulting in a non-zero amount returned from the  `Vault::underlyingTvl` for that particular token as the overall tvl is calculated in the Tvl modules :

```solidity
ERC20TvlModule.sol
function tvl(
      address vault
  ) external view noDelegateCall returns (Data[] memory data) {
      address[] memory tokens = IVault(vault).underlyingTokens();
      data = new Data[](tokens.length);
      for (uint256 i = 0; i < tokens.length; i++) {
          data[i].token = tokens[i];
          data[i].underlyingToken = tokens[i];
          data[i].amount = IERC20(tokens[i]).balanceOf(vault);
          data[i].underlyingAmount = data[i].amount;//@audit 
      }
  }
```

```solidity
DefaultBondTvlModule.sol
function tvl(
    address vault 
) external view noDelegateCall returns (Data[] memory data) {
    bytes memory data_ = vaultParams[vault];
    if (data_.length == 0) return data;
    address[] memory bonds = abi.decode(data_, (address[]));
    data = new Data[](bonds.length);
    for (uint256 i = 0; i < bonds.length; i++) {
        data[i].token = bonds[i];
        data[i].underlyingToken = IBond(bonds[i]).asset();
        data[i].amount = IERC20(bonds[i]).balanceOf(vault);
        data[i].underlyingAmount = data[i].amount;//@audit
    }
}
```

Here `underlyingTokens` and `underlyingAmount` returned from both Tvl contracts will be equal to the removed token.

Furthermore removal of this token will be **permanent** because this token will not be able to be transferred out, since it is not from a regular deposit and sender even non-malicious will not have the `lpAmount` to trigger withdrawal.

## Impact

- `Vault::removeToken` permanently dossed with 1 wei

## Code Snippet

https://github.com/mellow-finance/mellow-lrt/blob/ba168622a53e66c7655df5a6249760ecd9aa8f7d/src/Vault.sol#L205-L224

## Tool used

Manual Review

## Recommendation

Expose function to sweep tokens directly send to the Vault.