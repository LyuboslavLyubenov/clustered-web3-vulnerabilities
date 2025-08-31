# Cluster -1262

**Rank:** #248  
**Count:** 38  

## Label
Insufficient ownership and authorization checks enable unauthorized asset manipulation, leading to loss of control, unauthorized transfers, and potential theft through improper validation of caller identity or transaction order.

## Cluster Information
- **Total Findings:** 38

## Examples

### Example 1

**Auto Label:** Missing access controls and state management in withdrawal flows allow unauthorized operations, state inconsistencies, and fund loss during pause, cancellation, or buffer handling.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** High

## Description

The `StakingManager` contract implements a buffer system intended to maintain a reserve of HYPE tokens for processing withdrawals. However, the current implementation fails to utilize this buffer effectively, potentially forcing unnecessary validator exits and creating systemic risks during periods of high withdrawal demand.

The contract maintains two buffer-related state variables:

- `hypeBuffer`: The current amount of HYPE tokens held in reserve
- `targetBuffer`: The desired buffer size

When users stake HYPE, the contract attempts to maintain this buffer through the `_distributeStake()` function, which prioritizes filling the buffer before delegating remaining tokens to validators. However, the critical flaw lies in the withdrawal logic.

In `queueWithdrawal()`, the contract immediately initiates a validator withdrawal through `_withdrawFromValidator()` without first attempting to service the withdrawal from the buffer. This defeats the purpose of maintaining a buffer and can lead to:

1. Unnecessary validator exits even when sufficient funds are available in the buffer
2. Reduced staking efficiency as funds may be pulled from productive validation

```solidity
_withdrawFromValidator(currentDelegation, amount);
```

This contrasts with more robust implementations like Lido's buffered ETH system, where withdrawals are first serviced from the buffer, and validator exits are only triggered when withdrawal demands exceed available reserves.

### Proof of Concept

1. User A stakes 100 HYPE, with `targetBuffer` set to 50 HYPE
    - 50 HYPE goes to buffer
    - 50 HYPE is delegated to validator
2. User B requests withdrawal of 40 HYPE
    - Despite having 50 HYPE in buffer, contract calls `_withdrawFromValidator()`

## Recommendations

Modify the `queueWithdrawal()` function to prioritize using the buffer before forcing validator exits.

---
### Example 2

**Auto Label:** Missing access controls and state management in withdrawal flows allow unauthorized operations, state inconsistencies, and fund loss during pause, cancellation, or buffer handling.  

**Original Text Preview:**

<https://github.com/code-423n4/2025-04-kinetiq/blob/7f29c917c09341672e73be2f7917edf920ea2adb/src/StakingManager.sol# L919>

### Finding Description and Impact

**Root Cause**

The [cancelWithdrawal function](https://github.com/code-423n4/2025-04-kinetiq/blob/7f29c917c09341672e73be2f7917edf920ea2adb/src/StakingManager.sol# L919) in `StakingManager.sol` does not properly restore the contract’s internal state when a withdrawal request is canceled. Specifically, during a withdrawal initiated via `queueWithdrawal`, the `_withdrawFromValidator` function may:

* Deduct from the `hypeBuffer` state variable if sufficient liquidity is available.
* Append a `PendingOperation` to the `_pendingWithdrawals` array for any remaining amount to be processed on L1 if `hypeBuffer` is insufficient.

When a manager calls `cancelWithdrawal` to cancel a user’s withdrawal request, the function:

* Refunds the user’s kHYPE (including fees).
* Deducts the withdrawal amount from `totalQueuedWithdrawals`.
* Deletes the withdrawal request from `_withdrawalRequests`.

However, it fails to:

* Restore the `hypeBuffer` to its pre-withdrawal value.
* Remove or invalidate the corresponding `PendingOperation` (if any) from `_pendingWithdrawals`.

This leads to inconsistent accounting of the protocol’s liquidity and pending operations.

**Impact**

The failure to restore `hypeBuffer` and `_pendingWithdrawals` has the following consequences:

1. **Underreported Liquidity in `hypeBuffer`**:

   * The `hypeBuffer` remains lower than its actual value after cancellation, falsely indicating reduced on-chain liquidity.
   * This can force subsequent withdrawal requests to queue L1 operations unnecessarily, increasing delays for users and degrading user experience.
2. **Invalid Operations in `_pendingWithdrawals`**:

   * Any `PendingOperation` added to `_pendingWithdrawals` for a canceled withdrawal remains in the array and may be executed later via `executeL1Operations`.
   * This results in unnecessary withdrawals from L1 validators, which can:
   * Disrupt staking balances, potentially reducing staking rewards.
   * Increase gas costs for L1 interactions.
   * Incorrectly inflate `hypeBuffer` when L1 withdrawals are completed, leading to further accounting discrepancies.
3. **Accounting Inconsistency**:

   * The protocol’s internal state becomes misaligned, which may lead to suboptimal operational decisions, such as limiting withdrawals due to perceived low liquidity.
   * Over time, repeated cancellations without state restoration could accumulate errors, exacerbating liquidity mismanagement.

While this issue does not directly result in asset loss, it impairs protocol efficiency, increases operational costs, and may indirectly affect staking performance if L1 balances are disrupted.

### Recommended Mitigation Steps

To address this issue, the `cancelWithdrawal` function should be modified to fully revert the state changes made during `queueWithdrawal`. The following steps are recommended:

1. **Track Buffer Usage in `WithdrawalRequest`**:

   * Add a `bufferUsed` field to the `WithdrawalRequest` struct to record the amount deducted from `hypeBuffer`:

     
```

     struct WithdrawalRequest {
      uint256 hypeAmount;
      uint256 kHYPEAmount;
      uint256 kHYPEFee;
      uint256 bufferUsed; // Amount deducted from hypeBuffer
      uint256 timestamp;
     }
     
```

   * In `_withdrawFromValidator`, update the `bufferUsed` field:

     
```

     function _withdrawFromValidator(address validator, uint256 amount, OperationType operationType) internal {
      if (amount == 0) {
          return;
      }
      if (hypeBuffer >= amount) {
          hypeBuffer -= amount;
          _withdrawalRequests[msg.sender][nextWithdrawalId[msg.sender] - 1].bufferUsed = amount;
          return;
      }
      uint256 amountFromBuffer = hypeBuffer;
      _withdrawalRequests[msg.sender][nextWithdrawalId[msg.sender] - 1].bufferUsed = amountFromBuffer;
      uint256 remainingAmount = amount - amountFromBuffer;
      hypeBuffer = 0;
      if (remainingAmount > 0) {
          _pendingWithdrawals.push(PendingOperation({
              validator: validator,
              amount: remainingAmount,
              operationType: operationType
          }));
      }
      emit WithdrawalFromValidator(address(this), validator, amount, operationType);
     }
     
```

2. **Restore `hypeBuffer` in `cancelWithdrawal`**:

   * Modify `cancelWithdrawal` to restore `hypeBuffer` using the `bufferUsed` value:

     
```

     function cancelWithdrawal(address user, uint256 withdrawalId) external onlyRole(MANAGER_ROLE) {
      WithdrawalRequest storage request = _withdrawalRequests[user][withdrawalId];
      require(request.hypeAmount > 0, "Invalid withdrawal request");

      uint256 refundAmount = request.kHYPEAmount + request.kHYPEFee;

      // Restore hypeBuffer
      hypeBuffer += request.bufferUsed;

      totalQueuedWithdrawals -= request.hypeAmount;
      delete _withdrawalRequests[user][withdrawalId];

      kHYPE.transfer(user, refundAmount);

      emit WithdrawalCancelled(address(this), user, withdrawalId);
     }
     
```

3. **Handle `_pendingWithdrawals`**:

   * Tracking and removing specific operations from `_pendingWithdrawals` is complex due to its array structure. A simpler approach is to add a `withdrawalId` and `user` to `PendingOperation` to associate operations with withdrawal requests:

     
```

     struct PendingOperation {
      address validator;
      uint256 amount;
      OperationType operationType;
      address user;
      uint256 withdrawalId;
     }
     
```

   * Update `_withdrawFromValidator` to include these fields:

     
```

     if (remainingAmount > 0) {
      _pendingWithdrawals.push(PendingOperation({
          validator: validator,
          amount: remainingAmount,
          operationType: operationType,
          user: msg.sender,
          withdrawalId: nextWithdrawalId[msg.sender] - 1
      }));
     }
     
```

   * In `cancelWithdrawal`, mark or remove the operation:

     
```

     function cancelWithdrawal(address user, uint256 withdrawalId) external onlyRole(MANAGER_ROLE) {
      WithdrawalRequest storage request = _withdrawalRequests[user][withdrawalId];
      require(request.hypeAmount > 0, "Invalid withdrawal request");

      uint256 refundAmount = request.kHYPEAmount + request.kHYPEFee;

      // Restore hypeBuffer
      hypeBuffer += request.bufferUsed;

      // Remove associated pending withdrawal
      for (uint256 i = 0; i < _pendingWithdrawals.length; i++) {
          if (_pendingWithdrawals[i].user == user && _pendingWithdrawals[i].withdrawalId == withdrawalId) {
              _pendingWithdrawals[i] = _pendingWithdrawals[_pendingWithdrawals.length - 1];
              _pendingWithdrawals.pop();
              break;
          }
      }

      totalQueuedWithdrawals -= request.hypeAmount;
      delete _withdrawalRequests[user][withdrawalId];

      kHYPE.transfer(user, refundAmount);

      emit WithdrawalCancelled(address(this), user, withdrawalId);
     }
     
```

   * Alternatively, add a `cancelled` flag to `PendingOperation` and skip cancelled operations in `executeL1Operations`.

### Proof of Concept
```

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/StakingManager.sol";
import "../src/KHYPE.sol";

contract StakingManagerTest is Test {
    StakingManager stakingManager;
    KHYPE kHYPE;
    address user = address(0x123);
    address manager = address(0x456);
    address validator = address(0x789);
    uint256 constant HYPE_AMOUNT = 100 * 1e8; // 100 HYPE in 8 decimals
    uint256 constant BUFFER_INITIAL = 50 * 1e8; // 50 HYPE in 8 decimals

    function setUp() public {
        // Deploy contracts
        kHYPE = new KHYPE();
        stakingManager = new StakingManager();

        // Initialize contracts (simplified)
        kHYPE.initialize("Kinetiq HYPE", "kHYPE", manager, address(stakingManager), address(stakingManager), address(0x1));
        stakingManager.initialize(address(kHYPE), address(0x2), address(0x3), address(0x4), 10); // unstakeFeeRate = 10 basis points

        // Grant roles
        vm.prank(manager);
        stakingManager.grantRole(stakingManager.MANAGER_ROLE(), manager);

        // Setup initial state
        vm.deal(address(stakingManager), BUFFER_INITIAL);
        vm.store(address(stakingManager), bytes32(uint256(keccak256("hypeBuffer"))), bytes32(BUFFER_INITIAL));
        vm.prank(address(0x2)); // Mock ValidatorManager
        stakingManager.setDelegation(address(stakingManager), validator);

        // Mint kHYPE to user
        vm.prank(address(stakingManager));
        kHYPE.mint(user, HYPE_AMOUNT);
    }

    function testCancelWithdrawalStateInconsistency() public {
        // Step 1: User requests withdrawal
        vm.prank(user);
        stakingManager.queueWithdrawal(HYPE_AMOUNT);

        // Verify state after withdrawal request
        uint256 hypeBufferAfter = uint256(vm.load(address(stakingManager), bytes32(uint256(keccak256("hypeBuffer")))));
        assertEq(hypeBufferAfter, 0, "hypeBuffer should be 0 after withdrawal");
        // Note: Foundry doesn't directly support array length checks easily, assume _pendingWithdrawals has 1 entry

        // Step 2: Manager cancels withdrawal
        vm.prank(manager);
        stakingManager.cancelWithdrawal(user, 0);

        // Verify state after cancellation
        hypeBufferAfter = uint256(vm.load(address(stakingManager), bytes32(uint256(keccak256("hypeBuffer")))));
        assertEq(hypeBufferAfter, 0, "hypeBuffer incorrectly remains 0");
        // Expected: hypeBuffer should be 50 * 1e8
        // _pendingWithdrawals still contains an operation for 49.9 * 1e8

        // Step 3: Simulate impact
        // Another withdrawal would unnecessarily queue to L1 due to zero hypeBuffer
        vm.prank(user);
        kHYPE.mint(user, HYPE_AMOUNT); // Simulate user getting kHYPE again
        vm.prank(user);
        stakingManager.queueWithdrawal(HYPE_AMOUNT);

        // Check that a new pending withdrawal is queued
        // Note: Requires additional logic to verify _pendingWithdrawals length
    }
}
```

**Notes on PoC**

* The test demonstrates that `hypeBuffer` remains zero after cancellation, when it should be restored to `50 * 1e8`.
* Checking `_pendingWithdrawals` in Foundry is trickier due to array access limitations; additional helper functions or events may be needed to verify its state.
* The test assumes a simplified setup; real-world testing should include mocks for `ValidatorManager`, `StakingAccountant`, and L1 interactions.

**Kinetiq acknowledged**

---

---
### Example 3

**Auto Label:** Missing access controls and state management in withdrawal flows allow unauthorized operations, state inconsistencies, and fund loss during pause, cancellation, or buffer handling.  

**Original Text Preview:**

<https://github.com/code-423n4/2025-04-kinetiq/blob/7f29c917c09341672e73be2f7917edf920ea2adb/src/StakingManager.sol# L302-L312>

### Vulnerability Details

The [StakingManager::confirmWithdrawal()](https://github.com/code-423n4/2025-04-kinetiq/blob/7f29c917c09341672e73be2f7917edf920ea2adb/src/StakingManager.sol# L302-L312) function does not include the [whenWithdrawalNotPaused](https://github.com/code-423n4/2025-04-kinetiq/blob/7f29c917c09341672e73be2f7917edf920ea2adb/src/StakingManager.sol# L117-L120) modifier, despite being a withdrawal operation. According to the natspec docs of [pauseWithdrawal()](https://github.com/code-423n4/2025-04-kinetiq/blob/7f29c917c09341672e73be2f7917edf920ea2adb/src/StakingManager.sol# L898-L904), it should pause all withdrawal operations:
```

/**
 * @notice Pause withdrawal operations
 */
function pauseWithdrawal() external onlyRole(MANAGER_ROLE) {
    withdrawalPaused = true;
    emit WithdrawalPaused(msg.sender);
}

function confirmWithdrawal(uint256 withdrawalId) external nonReentrant whenNotPaused {
    uint256 amount = _processConfirmation(msg.sender, withdrawalId);
    require(amount > 0, "No valid withdrawal request");
    // ...
}
```

This inconsistency allows users to complete their withdrawal process by calling `confirmWithdrawal()` even when withdrawals are paused by the protocol. This defeats the purpose of the withdrawal pause functionality which is meant to halt all withdrawal-related operations during critical protocol conditions.

### Impact

Users can bypass withdrawal restrictions by confirming existing withdrawal requests during pause

### Proof of Concept

1. Protocol identifies suspicious activity and calls `pauseWithdrawal()`
2. User with pending withdrawal request calls `confirmWithdrawal()`
3. The withdrawal succeeds despite protocol pause, since missing modifier allows execution

### Recommended Mitigation Steps

Add withdrawal pause modifier
```

- function confirmWithdrawal(uint256 withdrawalId) external nonReentrant whenNotPaused {
+ function confirmWithdrawal(uint256 withdrawalId) external nonReentrant whenNotPaused whenWithdrawalNotPaused {
    uint256 amount = _processConfirmation(msg.sender, withdrawalId);
    // ...
}
```

**Kinetiq acknowledged**

---

---
