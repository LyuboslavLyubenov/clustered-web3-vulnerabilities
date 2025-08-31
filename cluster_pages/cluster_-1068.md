# Cluster -1068

**Rank:** #368  
**Count:** 15  

## Label
Inconsistent state maintenance due to flawed iteration, removal, or array management leads to incorrect state representation, misleading outputs, or unbounded loops, compromising data integrity and system reliability.

## Cluster Information
- **Total Findings:** 15

## Examples

### Example 1

**Auto Label:** Inconsistent state maintenance due to flawed iteration, removal, or array management leads to incorrect state representation, misleading outputs, or unbounded loops, compromising data integrity and system reliability.  

**Original Text Preview:**

##### Description

The receivers global state variable in MergeMarketplaceStrategy is of type `EnumerableSetUpgradeable`. If the order of an `EnumerableSet` is required, removing items in a loop using `at()` and `remove()` corrupts this order. This finding has been reported as informational because the developer team indicated that order is not currently important, but it is important to have this in mind for future reference.

##### BVSS

[AO:A/AC:M/AX:M/R:N/S:U/C:N/A:N/I:L/D:N/Y:N (1.1)](/bvss?q=AO:A/AC:M/AX:M/R:N/S:U/C:N/A:N/I:L/D:N/Y:N)

##### Recommendation

Consider using a different data structure or removing items by collecting them during the loop, then removing after the loop.

##### Remediation

**ACKNOWLEDGED:** The **B14G team** acknowledged this finding.

---
### Example 2

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
### Example 3

**Auto Label:** Inconsistent state management due to improper array indexing, missing deletions, or flawed iterator/ordering, leading to data corruption, incorrect state transitions, and erroneous operations.  

**Original Text Preview:**

##### Description
There is an issue at the line: https://github.com/MetaLex-Tech/borg-core/blob/9074503d37cfa1d777ef16f6c88b84c98b4f54eb/src/implants/daoVetoGrantImplant.sol#L199. There can be a case when the `proposalIndex != lastProposalIndex` check doesn't pass, and we remove the last proposal from the `currentProposals` at line https://github.com/MetaLex-Tech/borg-core/blob/9074503d37cfa1d777ef16f6c88b84c98b4f54eb/src/implants/daoVetoGrantImplant.sol#203 instead of removing an actual one. 

Let's assume there are two proposals in the `currentProposals` array with an ids `1` and `2`. If we try to delete the proposal with `id = 1` (the first element in the array), then `proposalIndex` would be equal to `1` and `lastProposalIndex = currentProposals.length - 1 = 1`. The `proposalIndex != lastProposalIndex`  check won't pass and the proposal which is being removed won't be replaced by the one at the end of the proposals array. After the `if` block, the last element is removed from the `currentProposals` array. Here the proposal with `id = 2` was deleted. 

This issue can lead to deleting (after vetoing) the incorrect proposal, so that an unwanted one can still be executed.

##### Recommendation
We recommend changing the check to `proposalIndex - 1 != lastProposalIndex` so that it will account for the difference between proposal index and array index.

---
