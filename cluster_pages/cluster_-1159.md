# Cluster -1159

**Rank:** #341  
**Count:** 19  

## Label
Failure to validate or enforce vesting parameters allows attackers or admins to manipulate token release schedules, leading to incorrect distributions, loss of economic control, or unintended early releases.

## Cluster Information
- **Total Findings:** 19

## Examples

### Example 1

**Auto Label:** Failure to validate timestamps against current time enables past or invalid events, allowing premature token claims, bypassing vesting schedules, and enabling unauthorized early access or fund withdrawals.  

**Original Text Preview:**

## Description
When a pending deallocation is slashed, the value of the deallocation may round down to zero. Therefore, an operator is able to call `modifyAllocations()` to update this deallocation. As a result, the `deallocationQueue` may no longer be sorted based on `effectBlock`.

The root cause of the issue is that when `slashOperator()` is called, if there is a pending deallocation, it will be slashed. That slashed amount may result in the `allocation.pendingDiff` rounding down to zero. This will not clear the pending deallocation from the `deallocationQueue`.

## AllocationManager.sol::_slashOperator()
```solidity
if (allocation.pendingDiff < 0) {
    uint64 slashedPending =
        uint64(uint256(uint128(-allocation.pendingDiff)).mulWadRoundUp(params.wadsToSlash[i]));
    allocation.pendingDiff += int128(uint128(slashedPending)); // @audit may be zeroed
    // ...
}
```

Once `allocation.pendingDiff` equals zero, the pending deallocation can be modified as the following check now passes:

## AllocationManager.sol::modifyAllocations()
```solidity
function modifyAllocations(
    address operator,
    AllocateParams[] memory params
) external onlyWhenNotPaused(PAUSED_MODIFY_ALLOCATIONS) {
    // ...
    for (uint256 i = 0; i < params.length; i++) {
        // ...
        for (uint256 j = 0; j < params[i].strategies.length; j++) {
            (StrategyInfo memory info, Allocation memory allocation) =
                _getUpdatedAllocation(operator, operatorSet.key(), strategy);
            // @audit modification no longer counts as pending even though effectBlock has not passed
            require(allocation.pendingDiff == 0, ModificationAlreadyPending());
            // ...
        }
    }
}
```

`allocation.pendingDiff == 0` incorrectly assumes that the allocation has no pending modifications, even though the `allocation.effectBlock` has not passed. Once this allocation is modified, the `allocation.effectBlock` will be set again, resulting in the `deallocationQueue` being unsorted.

An unsorted `deallocationQueue` will result in the `_clearDeallocationQeue()` function not clearing all of the deallocations in the queue. `getAllocatableMagnitude()` will also incorrectly calculate the return value as it discounts deallocations.

## Recommendations
Consider changing the function `modifyAllocations()` to check for pending modifications based on `allocation.effectBlock` instead of `allocation.pendingDiff`.

## Resolution
The EigenLayer team has implemented the recommended fix above. This issue has been resolved in PR #1052.

---
### Example 2

**Auto Label:** Failure to validate timestamps against current time enables past or invalid events, allowing premature token claims, bypassing vesting schedules, and enabling unauthorized early access or fund withdrawals.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

The balance of the user's validators can be updated by creating a new snapshot and proving the balance of the validators. If a user doesn't start a new snapshot in time then anyone can start a snapshot for that user and update the user's balance. Users have an incentive to not update their balances if they get slashed in the beacon chain. A user can block the snapshot finalization and snapshot creation process and stop balance updates by performing this attack. The issue is that the `validateSnapshotProofs()` code doesn't allow updating the validators balance if the validator balance is updated after the snapshot creation:

```solidity
            if (validatorDetails.lastBalanceUpdateTimestamp >= node.currentSnapshotTimestamp) {
                revert ValidatorAlreadyProved();
            }
```

And a snapshot only finalizes when the `remainingProofs` is 0. So This is the attack POC:

1. User1 who has slashed on the beacon chain wants to prevent balance updates.
2. User1 would create new validator and call `validateWithdrawalCredentials()` and `startSnapshot()` add the same Ethereum block.
3. As a result for the new validator: `validatorDetails.lastBalanceUpdateTimestamp` would equal to `node.currentSnapshotTimestamp`.
4. Because a new validator has been counted on the `remainingProofs` and it's not possible to prove its balance for the current snapshot then `remainingProofs` would never reach 0 and the current snapshot would never finish and the balance update process would be broken.

## Recommendations

It's best to change `validateWithdrawalCredentials()` and not allow the creation of the new validators for the current block's timestamp.
Changing the above condition to `validatorDetails.lastBalanceUpdateTimestamp > node.currentSnapshotTimestamp` in `validateSnapshotProofs()` will create another issue.

---
### Example 3

**Auto Label:** Failure to validate or enforce vesting parameters allows attackers or admins to manipulate token release schedules, leading to incorrect distributions, loss of economic control, or unintended early releases.  

**Original Text Preview:**

## Context
See below.

## Description
The farm scripts check several assumptions on the involved contracts. Furthermore, a `Phase1b_UsdsSkyFarmingCheckScript` script was created that should do a sanity check on a deployed instance. Currently, the sanity checks are incomplete; the missing pieces are: `Vest.cap`, `Vest.tau`, `Vest.bgn`, and `Vest.tot`.

## Recommendation
Consider adding additional checks:
- `UsdsSkyFarmingInit.sol#L74`: `p.vestBgn` should be the start date minus the rewards duration, otherwise, the initial rewards rate differs from the mint rate. Then, users should already be able to stake prior to the start date; otherwise, initial tokens will be lost when distributed to a farm without stakers.

- Add the following checks within the `Phase1b_UsdsSkyFarmingCheckScript`:
  ```solidity
  require(DssVestWithGemLike(vest).cap() == {vestCap}, "DssVest/invalid-cap");
  require(DssVestWithGemLike(vest).bgn(vestId) == vestBgn, "DssVest/invalid-bgn");
  require(DssVestWithGemLike(vest).tot(vestId) == vestTot, "DssVest/invalid-tot");
  require(DssVestWithGemLike(vest).fin(vestId) == vestBgn + vestTau, "DssVest/invalid-tau");
  ```

## Maker
We decided that the ownership of the `DssVest` contract lies outside this repo, so any contract-level parameter checks are out of the scope.

## Cantina Managed
The `Phase1b_UsdsSkyFarmingCheckScript` script now checks `vest.bgn`, `vest.fin`, and `vest.tot` against pre-configured values. Further checks on the cap and reward rates will be performed in the main spell.

---
