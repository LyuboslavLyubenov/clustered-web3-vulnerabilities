# Cluster -1126

**Rank:** #352  
**Count:** 17  

## Label
Inconsistent state management due to improper array indexing, missing deletions, or flawed iterator/ordering, leading to data corruption, incorrect state transitions, and erroneous operations.

## Cluster Information
- **Total Findings:** 17

## Examples

### Example 1

**Auto Label:** Inconsistent state management due to improper array indexing, missing deletions, or flawed iterator/ordering, leading to data corruption, incorrect state transitions, and erroneous operations.  

**Original Text Preview:**

## Lock Removal in SmartTable

When a lock is removed, it is not actually deleted from the SmartTable storing lock records. This renders the data accessible in the system even after it is supposedly removed. Since locks are not fully removed from the SmartTable, view functions may show locks that should have been deleted. 

Also, the `lock_index` is not validated to ensure it is within the bounds of the investor’s lock count. Thus, the `lock_index` values may be out of bounds, potentially attempting to delete nonexistent records. As a result, the same lock may be removed multiple times repeatedly, each time decreasing the lock count.

## Remediation

- Check that `lock_index <= total_investor_count` in both the view functions and the removal function.

## Patch

Resolved in 06538b8.

---
### Example 2

**Auto Label:** Inconsistent state management due to improper array indexing, missing deletions, or flawed iterator/ordering, leading to data corruption, incorrect state transitions, and erroneous operations.  

**Original Text Preview:**

##### Description
There is an issue at the line: https://github.com/MetaLex-Tech/borg-core/blob/9074503d37cfa1d777ef16f6c88b84c98b4f54eb/src/implants/daoVetoGrantImplant.sol#L199. There can be a case when the `proposalIndex != lastProposalIndex` check doesn't pass, and we remove the last proposal from the `currentProposals` at line https://github.com/MetaLex-Tech/borg-core/blob/9074503d37cfa1d777ef16f6c88b84c98b4f54eb/src/implants/daoVetoGrantImplant.sol#203 instead of removing an actual one. 

Let's assume there are two proposals in the `currentProposals` array with an ids `1` and `2`. If we try to delete the proposal with `id = 1` (the first element in the array), then `proposalIndex` would be equal to `1` and `lastProposalIndex = currentProposals.length - 1 = 1`. The `proposalIndex != lastProposalIndex`  check won't pass and the proposal which is being removed won't be replaced by the one at the end of the proposals array. After the `if` block, the last element is removed from the `currentProposals` array. Here the proposal with `id = 2` was deleted. 

This issue can lead to deleting (after vetoing) the incorrect proposal, so that an unwanted one can still be executed.

##### Recommendation
We recommend changing the check to `proposalIndex - 1 != lastProposalIndex` so that it will account for the difference between proposal index and array index.

---
### Example 3

**Auto Label:** Inconsistent state management due to improper array indexing, missing deletions, or flawed iterator/ordering, leading to data corruption, incorrect state transitions, and erroneous operations.  

**Original Text Preview:**

**Description:** The function attempts to remove the withdrawal at index `0`, while it uses the withdrawal at index `i` to call `completeQueuedWithdrawal()`. Since each withdrawal can only be completed once, the `delayedEffectiveBalanceQueue[]` list will eventually contain withdrawals that have already been completed. When the function tries to complete a withdrawal that has already been completed, it invariably reverts.

```solidity
for (uint256 i; i < delayedEffectiveBalanceQueue.length; i++) {
    IDelegationManager.Withdrawal memory withdrawal = delayedEffectiveBalanceQueue[i];
    if (uint32(block.number) - withdrawal.startBlock > withdrawalDelay) {
        delayedEffectiveBalanceQueue.remove(0); // @audit Remove withdrawal at index 0
        claimedEffectiveBalance += withdrawal.shares[0];
        eigenDelegationManager.completeQueuedWithdrawal(withdrawal, tokens, 0, true); // @audit Complete withdrawal of index i
    } else {
        break;
    }
}
```

**Impact:** The `claimEffectiveBalance()` function consistently reverts, making it impossible to complete queue withdrawals and therefore locking ETH.

**Proof of Concept:** Consider the following scenario:

1. Initially, the `delayedEffectiveBalanceQueue[]` list includes five withdrawals `[a, b, c, d, e]`.
2. The `claimEffectiveBalance()` function is called.
    - In the first loop iteration `i = 0`, withdrawal `a` is removed and completed. The list now becomes `[b, c, d, e]`.
    - In the second loop iteration `i = 1`, withdrawal `b` is removed, but withdrawal `c` is completed. The list now becomes `[c, d, e]`.
    - In the third loop iteration `i = 2`, the function checks withdrawal `e` and assumes the withdrawal delay has not yet been reached. The loop breaks at this point and the function stops.
3. The next time the `claimEffectiveBalance()` function is called.
    - In the first loop iteration `i = 0`, the function tries to remove and complete withdrawal `c`. However, since withdrawal `c` has already been completed, the call to `completeQueuedWithdrawal()` will revert.

**Recommended Mitigation:** Consider using a consistent index for checking, removing and completing withdrawals.

**Casimir:**
Fixed in [35fdf1e](https://github.com/casimirlabs/casimir-contracts/commit/35fdf1e42ad2a38f47028a8468efc0e78e6e7f67)

**Cyfrin:** Verified.

---
