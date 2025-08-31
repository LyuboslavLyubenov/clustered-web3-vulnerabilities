# Cluster -1023

**Rank:** #241  
**Count:** 39  

## Label
Unauthorized access to user funds through insecure delegation and lack of access control boundaries, enabling malicious actors to drain assets without user consent or protocol oversight.

## Cluster Information
- **Total Findings:** 39

## Examples

### Example 1

**Auto Label:** Unauthorized fund withdrawal via insufficient access controls and flawed claim mechanisms, enabling owner-driven extraction of user entitlements or locked funds.  

**Original Text Preview:**

## Severity

Medium Risk

## Description

In Pip.sol, when a user deposits, they receive a nullifier code which is required when
withdrawing from a different address. However, since it is possible for a user to forget their nullifier
code, the funds could become permanently stuck in the contract.


## Recommendation

Add a rescueFunds function along with an on-chain view function to check
whether the nullifier hash (used during withdrawal) has been consumed. This would enable the
protocol to verify that the user genuinely forgot their nullifier code and facilitate the recovery of their
funds.

## Team Response

Acknowledged.

---
### Example 2

**Auto Label:** Unauthorized asset withdrawal due to flawed authorization and state inconsistency, enabling fund drainage and loss of user-owned assets.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** High

## Description

When a user adds liquidity to the pool, the system triggers the `_updateLockData` function, which in turn calls the `Fenwicks._deposit` function to record the deposit amount and its associated lock duration (e.g., 7 days). The `Fenwicks._deposit` function updates a data structure known as the Fenwick tree to manage and lock the deposit amounts over time.

For a user's first deposit (`isDirty=true`), the system resets the tree values to avoid incorrect data affecting subsequent deposits:

```solidity
function _fenwicksUpdate(FenwicksData storage self, uint256 maxDepositEntries, uint256 treeIndex, uint256 value)
    internal
{
    // when registering new values after full withdrawal, we need to overwrite dirty values
    bool isDirty = self.sortedKeys.length == 1;
    while (treeIndex <= maxDepositEntries) {
        // when dirty overwrite, otherwise add
@1>      isDirty ? self.fenwicksTree[treeIndex] = value : self.fenwicksTree[treeIndex] += value;
@2>      uint256 nextIndex = treeIndex + (treeIndex & (~treeIndex + 1));
        if (nextIndex > maxDepositEntries) break; // Stop if next update would exceed maxDepositEntries
        treeIndex = nextIndex;
    }
}
```

The problem occurs because it only reset specific indices such as 1, 2, 4, 8, etc. and **does not reset other indices like 3, 5, 6, etc.**
After a full withdrawal, if the user previously had deposits at indices 3 and 5, they remain untouched. As a result, the old values from these indices are mistakenly added to new deposit amounts. Users can unlock more amounts and in fact, get (`removeLiquidity`) the funds of other users in the pool.

**Example Scenario:**

1. Initial Deposit:
   User deposits at indices 1, 2, and 3 updating the Fenwick tree accordingly.
2. Full Withdrawal:
   The user withdraws all funds (after the unlock period).
3. First Deposit After Withdrawal:
   The user deposits again, with isDirty=true. Indices 1, 2, 4, 8, 16, etc., are updated with the new values (resetted).
4. Second Deposit:
   The user deposits again, updating index 2 with the new value.
5. Third Deposit:
   The user deposits again, and index 3 is updated. However, because the old value from the initial deposit (step 1) remains in index 3, a new deposit value will be added to the old value.

Result:

Because the old value from the first deposits is incorrectly retained in index 3, the user ends up unlocking more funds than they actually deposited. This could allow the user to withdraw assets that they did not originally contribute, and potentially withdraw funds belonging to other users.

## Recommendations

Ensure that the system resets all indices in the Fenwick tree after a full withdrawal, not just specific indices like 1, 2, 4, 8, 16, etc.

---
### Example 3

**Auto Label:** Unauthorized access and uncontrolled asset manipulation leading to denial of liquidation, fund loss, or slippage through bypassed safeguards and direct ETH handling.  

**Original Text Preview:**

In the `claimExcess()` function of the `LiquidationBuffer` contract, the contract owner can claim excess buffer by redeeming shares from a specified vault. However, there is no check for a minimum amount of redeemed assets that should be acceptable to be received by the owner, which makes this function not protected against slippage, and thus receiving less redeemed assets than intended:

```solidity
  function claimExcess(uint8 _vaultId) external onlyOwner {
        uint256 excess = claimableExcess[_vaultId];
        claimableExcess[_vaultId] = 0;

        if (excess > 0) {
            IVault(POSITION_MANAGER.getVault(_vaultId).addr).redeem(excess, msg.sender, address(this));
        }
    }
```

Recommendation: add a `minAmountOut` parameter and check against it to mitigate slippage.

---
