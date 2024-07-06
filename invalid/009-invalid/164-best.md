Uneven Mango Ferret

Medium

# Incorrect TVL Calculation Due to Discrepancy Between Token Amount and Underlying Amount

## Summary
The DefaultBondTvlModule contract calculates the Total Value Locked (TVL) of a vault by considering the bond tokens and their underlying assets. However, there is a potential discrepancy between the bond token amounts and their underlying amounts, which can lead to incorrect TVL calculations
## Vulnerability Detail
The vulnerability arises from the assumption that the amount of bond tokens and the amount of underlying tokens are always equal. In the tvl function, the code currently sets both the amount and underlyingAmount to the balance of bond tokens

However, the balance of the underlying tokens can differ from the balance of the bond tokens, as bond tokens are contracts that represent underlying assets. An attacker can exploit this by manipulating the balance of the underlying asset, resulting in discrepancies between the reported amount and underlyingAmount.

For example, if an attacker sends extra underlying tokens directly to the vault, the underlyingAmount will not reflect this additional balance, leading to an incorrect TVL calculation. This can cause issues in various parts of the contract that rely on accurate TVL values, such as issuing tokens, minting, and burning evaluations.
## Impact
Incorrect TVL calculations for vaults.
Potential exploitation by attackers to manipulate TVL.
Inaccuracies in token issuance, minting, and burning processes.
Potential financial loss or misallocation of resources due to incorrect TVL values.
## Code Snippet
https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/modules/symbiotic/DefaultBondTvlModule.sol#L36

```solidity
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
            data[i].underlyingAmount = data[i].amount;
        }
    }
```
## Tool used

Manual Review

## Recommendation
Replace the line where the underlyingAmount is set to the bond token balance with the correct balance of the underlying asset. 

```diff
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
+         data[i].underlyingAmount =  IERC20(bonds[i].asset()).balanceOf(vault);
-          data[i].underlyingAmount = data[i].amount;
        }
    }
```