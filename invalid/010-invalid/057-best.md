Orbiting Mercurial Okapi

Medium

# Missing withdrawal request deadline check may prevent users from executing emergency withdrawals

## Summary
The magnitude of the withdrawal request deadline is not compared against the emergency withdrawal delay. This may lead to scenarios where a user claiming an emergency withdrawal is prevented from doing so because the protocol will automatically cancel the current withdrawal request.
## Vulnerability Detail
Users can decide the deadline at which their request will expire:

```javascript
    function registerWithdrawal(
        address to,
        uint256 lpAmount,
        uint256[] memory minAmounts,
        uint256 deadline,
@>      uint256 requestDeadline,
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
@>          deadline: requestDeadline,
            timestamp: timestamp
        });
        _withdrawalRequest[sender] = request;
        _pendingWithdrawers.add(sender);
        _transfer(sender, address(this), lpAmount);
        emit WithdrawalRequested(sender, request);
    }

```

 However the current code allows them to set a value smaller than `VaultConfigurator::_emergencyWithdrawalDelay.value`. This may lead to a scenario where the user wants to execute an emergency withdrawal and will not be able to do so because the following if statement will end the function call:

 ```javascript
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
@>      if (timestamp > request.deadline) {
@>          _cancelWithdrawalRequest(sender);
@>          return actualAmounts;
@>      }
...
 ``` 
## Impact
The users will have to file a new withdrawal request setting the desired paramaters and wait again for the `VaultConfigurator::_emergencyWithdrawalDelay.value` to pass before they can claim the emergency withdrawal. See PoC:

<details>

<summary>Code</summary>

Include in `VaultTest.t.sol`.

```javascript
    function testEmergencyWithdrawIsCancelled() external {
        Vault vault = new Vault("Mellow LRT Vault", "mLRT", admin);
        vm.startPrank(admin);
        vault.grantRole(vault.ADMIN_DELEGATE_ROLE(), admin);
        vault.grantRole(vault.OPERATOR(), operator);
        _setUp(vault);
        vm.stopPrank();
        _initialDeposit(vault);

        VaultConfigurator configurator = VaultConfigurator(
            address(vault.configurator())
        );

        vm.startPrank(admin);
        configurator.stageEmergencyWithdrawalDelay(2 * 7 * 24 * 60 * 60);
        configurator.commitEmergencyWithdrawalDelay();
        vm.stopPrank();

        address depositor = address(bytes20(keccak256("depositor")));
        vm.startPrank(depositor);
        deal(Constants.WSTETH, depositor, 10 ether);
        IERC20(Constants.WSTETH).safeIncreaseAllowance(
            address(vault),
            10 ether
        );
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 10 ether;
        uint256[] memory minAmounts = new uint256[](3);
        vault.deposit(depositor, amounts, 10 ether, type(uint256).max);
        vault.registerWithdrawal(
            depositor,
            10 ether,
            minAmounts,
            type(uint256).max,
            block.timestamp + 7 * 24 * 60 * 60,
            false
        );
        vm.recordLogs();
        vm.warp(block.timestamp + 2 weeks);
        vault.emergencyWithdraw(new uint256[](3), type(uint256).max);
        Vm.Log[] memory e = vm.getRecordedLogs();
        assertEq(e[1].emitter, address(vault));
        assertEq(e[1].topics.length, 2);
        assertEq(e[1].topics[0], IVault.WithdrawalRequestCanceled.selector);
        assertEq(e[1].topics[1], bytes32(uint256(uint160(depositor))));
        assertEq(e[1].data, abi.encode(tx.origin));
    }
```
</details>

## Code Snippet
https://github.com/sherlock-audit/2024-06-mellow/blob/main/mellow-lrt/src/Vault.sol#L384-L387

```javascript
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
@>      if (timestamp > request.deadline) {
@>          _cancelWithdrawalRequest(sender);
@>          return actualAmounts;
@>      }

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
## Tool used

Manual Review

## Recommendation
Add a check in `Vault::registerWithdrawal` to see if the chosen deadline is smaller than `VaultConfigurator::_emergencyWithdrawalDelay.value` + a safety margin acting as a minum threshold defining the time window after the request expired in which the user could claim the  emergency withdrawal, if so, revert with custom error. This will prevent users from being in a situation where they need their funds but find them unavailable. 

```diff
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
+       if (
+           requestDeadline <
+           block.timestamp +
+               configurator.emergencyWithdrawalDelay() +
+               SAFETY_MARGIN
+       ) revert Deadline();
...
```
