# Cluster -1380

**Rank:** #238  
**Count:** 41  

## Label
Insufficient state validation and incorrect time-based access controls lead to unauthorized actions, bid manipulation, and denial of critical operations, enabling exploitation of auction lifecycles and user permissions.

## Cluster Information
- **Total Findings:** 41

## Examples

### Example 1

**Auto Label:** Race conditions and improper state synchronization during critical transitions, leading to inconsistent epoch calculations, flawed timing logic, and unauthorized state changes.  

**Original Text Preview:**

## DefiAppHomeCenter Contract Review

## Context
- `DeFiAppHomeCenter.sol#L261`
- `DeFiAppHomeCenter.sol#L324`
- `DeFiAppHomeCenter.sol#L350`

## Summary
The contract contains an incorrect validation in `_setDefaultEpochDuration` where epoch duration is compared directly with `NEXT_EPOCH_BLOCKS_PREFACE` without accounting for block cadence conversion, leading to potential epoch initialization failures.

## Finding Description
In the `DefiAppHomeCenter` contract, there's a mismatch between how epoch duration is validated and how it's actually used:

In `_setDefaultEpochDuration`, the validation is:

```solidity
require(_epochDuration > NEXT_EPOCH_BLOCKS_PREFACE, DefiAppHomeCenter_invalidEpochDuration());
```

However, when this duration is used in `initializeNextEpoch`, it's converted to blocks using `BLOCK_CADENCE`:

```solidity
block.number + ($.defaultEpochDuration / BLOCK_CADENCE)
```

This converted value must then satisfy:

```solidity
require(endBlock > block.number + NEXT_EPOCH_BLOCKS_PREFACE, DefiAppHomeCenter_invalidEndBlock());
```

The issue is that `_epochDuration` is in seconds, while `NEXT_EPOCH_BLOCKS_PREFACE` is in blocks. The validation should compare equivalent units by either converting `NEXT_EPOCH_BLOCKS_PREFACE` to seconds or converting `_epochDuration` to blocks.

## Recommendation
Update the validation logic in `_setDefaultEpochDuration` to compare equivalent units.

## DeFi App
Fixed in commit `059fd13b`.

---
### Example 2

**Auto Label:** Race conditions and improper state synchronization during critical transitions, leading to inconsistent epoch calculations, flawed timing logic, and unauthorized state changes.  

**Original Text Preview:**

## Vulnerability Report

## Context
DeﬁAppHomeCenter.sol#L270-L289

## Description
The `initializeNextEpoch()` function in the DefiAppHomeCenter contract updates epochs for token distribution. Delaying the epoch update can cause a Denial of Service (DoS) when creating new epochs. During the calculation of the `endBlock`, it considers the last epoch's end block. This makes the value of `endBlock` close to the current `block.number`. As a result, the `_setEpochParams()` function will fail at the following line:

```solidity
require(endBlock > block.number + NEXT_EPOCH_BLOCKS_PREFACE, DefiAppHomeCenter_invalidEndBlock());
```

## Proof of Concept
```solidity
function test_skip_epoch() public {
    // Initialize epochs
    vm.prank(Admin.addr);
    center.initializeNextEpoch();
    
    // Delaying the update of next epoch will cause a DOS,
    // it needs to be close to the end of next epoch
    vm.roll(block.number + (53 days / 2));
    
    // Reverts with "panic: arithmetic underflow or overflow"
    vm.expectRevert();
    center.initializeNextEpoch();
}
```

## Recommendation
It is recommended to implement logic that processes missing epochs by iterating through them and initializing each one.

## DeFi App
Fixed in commit `1f28203f`.

---
### Example 3

**Auto Label:** Insufficient state validation and incorrect time-based access controls lead to unauthorized actions, bid manipulation, and denial of critical operations, enabling exploitation of auction lifecycles and user permissions.  

**Original Text Preview:**

## Context
Kimap.sol#L49-L64

## Description
The `_isValidName` function reverts with the errors `NameTooShort()`, `NameTooLong()`, and `InvalidNameCharacter()`. The function is used for determining valid names as well as facts and notes. An error referencing "names" might be confusing.

## Recommendation
Consider using `Label` rather than `Name` in error messages.

## Hyperware
Fixed in PR 4.

## Cantina Managed
Fixed.

---
