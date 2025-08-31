# Cluster -1251

**Rank:** #280  
**Count:** 29  

## Label
Failure to clear state upon ownership transfer enables persistent access or unintended control by former owners, leading to security breaches and loss of deterministic access.

## Cluster Information
- **Total Findings:** 29

## Examples

### Example 1

**Auto Label:** Failure to enforce runtime access controls during state transitions, enabling unauthorized or unintended state changes with high impact on data consistency and security.  

**Original Text Preview:**

## Storage Resource Modification

`storage_resource::split_by_epoch` modifies an existing storage object by splitting it into two separate objects. The original storage is modified to have `start_epoch → split_epoch`, while a new storage object is created to cover `split_epoch → end_epoch`. However, the accounting mechanism for storage reclamation does not properly adjust to this split, resulting in inconsistencies when `decrease_storage_to_reclaim` is called. 

Thus, `extend_blob` will not work because `decrease_storage_to_reclaim` in `storage_accounting` will try to decrease the storage size in the wrong epoch.

## Remediation

Ensure that storage reclamation is properly tracked for the new epoch by modifying `FutureAccounting` to store `used_capacity` for future epochs.

## Patch

Fixed in `f9a7865`.

---
### Example 2

**Auto Label:** Failure to enforce runtime access controls during state transitions, enabling unauthorized or unintended state changes with high impact on data consistency and security.  

**Original Text Preview:**

## Context
- **Usd0PP.sol**: Line 122
- **UsualX.sol**: Line 95

## Description
The `Usd0PP.Usd0PPStorageV0.unusedTreasuryAllocationRate` and `UsualX.UsualXStorageV0.unusedBurnRatioBps` are storage state variables which are used in the current on-chain contracts but won't be used anymore after the migration to new contract implementations. They are just kept there to preserve the storage layout of contracts.

Ideally, since these storage values won’t be used anymore, they should be reset to 0 during or before the migration. Keeping unused non-zero storage values is not ideal and can cause issues in future implementations of contracts if not handled carefully.

## Recommendation
Consider resetting `Usd0PP.Usd0PPStorageV0.unusedTreasuryAllocationRate` and `UsualX.UsualXStorageV0.unusedBurnRatioBps` to 0 values during migration.

## Usual
Acknowledged. Will be handled carefully (e.g., our namespace storage struct is append-only).

## Cantina Managed
Acknowledged.

---
### Example 3

**Auto Label:** Failure to enforce runtime access controls during state transitions, enabling unauthorized or unintended state changes with high impact on data consistency and security.  

**Original Text Preview:**

**Severity** : Low

**Status**: Acknowledged

**Description**: 

Users are still able to claim their vested tokens during migration mode, which might not be intended as it can disrupt ongoing migration activities.
 Allowing claims during migration can lead to potential inconsistencies, especially when trying to migrate to a new version of the contract.
 
**Recommendation**: 

Prevent claims during migration mode by adding a validation:
```solidity
require(!migrationMode, "Migration mode active, claiming is disabled");
```

---
