# Cluster -1204

**Rank:** #391  
**Count:** 13  

## Label
State inconsistency due to improper sequencing, validation, or immutability handling, leading to incorrect state exposure, transaction failures, or financial loss.

## Cluster Information
- **Total Findings:** 13

## Examples

### Example 1

**Auto Label:** State inconsistency due to improper sequencing, validation, or immutability handling, leading to incorrect state exposure, transaction failures, or financial loss.  

**Original Text Preview:**

## Context
(No context files were provided by the reviewer)

## Description/Recommendation
- **IMech.sol#L15-L21:** `maxDeliveryRate` and `paymentType` can be defined as `view`. This will guarantee that when those interface endpoints are used, `staticcall` is made instead of a regular call.
- **IStaking.sol:** `IStaking.sol` does not seem to be used and perhaps can be removed.

## Valory
Fixed on PR 94.

## Cantina Managed
Fix verified.

---
### Example 2

**Auto Label:** Inconsistent state validation leading to data loss or incorrect exposure of uninitialized settlement records.  

**Original Text Preview:**

##### Description

In the `getPrices` function of the **BeraAuctionHouse** contract, there is a contradictory logic when dealing with uninitialized settlement data. Specifically, if `settlementState.blockTimestamp == 0`, the function reverts with a `MissingSettlementsData` error. However, if `settlementState.winner == address(0)`, the function skips the iteration using `continue`. In practice, uninitialized data is likely to have both `blockTimestamp == 0` and `winner == address(0)`, which results in inconsistent handling: some cases halt execution while others are silently ignored. This inconsistency can lead to confusion and unintended behavior, particularly when the contract processes settlement history.

##### BVSS

[AO:A/AC:L/AX:M/R:N/S:U/C:N/A:N/I:L/D:N/Y:N (1.7)](/bvss?q=AO:A/AC:L/AX:M/R:N/S:U/C:N/A:N/I:L/D:N/Y:N)

##### Recommendation

Unify the handling of uninitialized or invalid auction data to avoid contradictory behavior. If encountering uninitialized data is a critical issue, revert execution for both conditions. If skipping such entries is acceptable, handle both conditions with a single `continue` statement. For example:

```
if (settlementState.blockTimestamp == 0 || settlementState.winner == address(0)) {
    continue; // Skip uninitialized or no-bid settlements
}
```

##### Remediation

**SOLVED:** The **Beranames team** solved the issue in the specified commit id.

##### Remediation Hash

<https://github.com/Beranames/beranames-contracts-v2/pull/104/commits/5676c96f5f4ac10c972fdec24b11e48c848d1366>

---
### Example 3

**Auto Label:** Inconsistent state validation leading to data loss or incorrect exposure of uninitialized settlement records.  

**Original Text Preview:**

##### Description

The `getSettlements(uint256)` and `getPrices` functions in the **BeraAuctionHouse** contract do not properly handle cases where `latestTokenId` is `0` and the last auction is not settled. When `latestTokenId` equals `0`, the functions will include it in the return value regardless of whether the auction has been settled. This behavior can lead to the inclusion of incomplete or invalid settlement data, which might mislead users or systems relying on this function for accurate historical auction information.

##### BVSS

[AO:A/AC:L/AX:M/R:N/S:U/C:N/A:N/I:L/D:N/Y:N (1.7)](/bvss?q=AO:A/AC:L/AX:M/R:N/S:U/C:N/A:N/I:L/D:N/Y:N)

##### Recommendation

Add a validation check to exclude the latest token ID if it corresponds to an unsettled auction. Specifically, ensure that the functions skip the token if `auctionStorage.settled` is `false` or if other indicators show that the auction has not been finalized. This adjustment ensures that only settled and valid auction data is included in the results.

##### Remediation

**SOLVED:** The **Beranames team** solved the issue in the specified commit id.

##### Remediation Hash

<https://github.com/Beranames/beranames-contracts-v2/pull/104/commits/87c0474f8bacb2fab95133d41edf62051ac82334>

##### References

src/auction/BeraAuctionHouse.sol#L343-L345, L384-L386

---
