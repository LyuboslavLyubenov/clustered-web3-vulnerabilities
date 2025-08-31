# Cluster -1286

**Rank:** #289  
**Count:** 26  

## Label
Failure to synchronize asset state and ownership transfers leads to irreversible loss of user control and inconsistent value reporting, enabling fund mismanagement and protocol failures.

## Cluster Information
- **Total Findings:** 26

## Examples

### Example 1

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
### Example 2

**Auto Label:** Failure to isolate user assets from operational funds enables unauthorized draining and permanent loss of user funds through unchecked access and lack of asset accountability.  

**Original Text Preview:**

```javascript
if (tokenState == TokenState.Trading && isTradingFromDex && !isPrizePool) {
                uint256 feeAmount = (value * feeBps) / 10000;
                uint256 protocolFee = feeAmount * protocolFeeBps / 10000;
                uint256 prizePoolFee = feeAmount - protocolFee;
                uint256 transferAmount = value - feeAmount;

                // Transfer fee to treasury and prize pool
                super._update(from, treasury, protocolFee);
                super._update(from, prizePool, prizePoolFee);

                // Transfer remainder to user
                super._update(from, to, transferAmount);
            }
```

In the \_update() function, the transfer fee is only applied to tokens traded on the Uniswap V2 pool. As a result, users can place buy or sell orders on Uniswap V3 at prices close to those on Uniswap V2, allowing them to trade without paying any transfer fees. This may be a known issue.

---
### Example 3

**Auto Label:** Unauthorized asset withdrawal due to flawed authorization and state inconsistency, enabling fund drainage and loss of user-owned assets.  

**Original Text Preview:**

*Note: This is based on the 2024-07 Loopfi audit [M-35](https://github.com/code-423n4/2024-07-loopfi-findings/issues/81) issue. This protocol team applied a fix, but the fix is incomplete.*

Only the bug in the `_onDeposit()` was fixed, but not the one in `_onWithdraw()`. `PositionAction4626.sol#_onWithdraw` does not withdraw from the correct position, it should withdraw from `position` instead of `address(this)`.

```solidity
    function _onDeposit(address vault, address position, address src, uint256 amount) internal override returns (uint256) {
        address collateral = address(ICDPVault(vault).token());

        // if the src is not the collateralToken, we need to deposit the underlying into the ERC4626 vault
        if (src != collateral) {
            address underlying = IERC4626(collateral).asset();
            IERC20(underlying).forceApprove(collateral, amount);
            amount = IERC4626(collateral).deposit(amount, address(this));
        }

        IERC20(collateral).forceApprove(vault, amount);

        // @audit-note: This was fixed.
        return ICDPVault(vault).deposit(position, amount);
    }


    function _onWithdraw(
        address vault,
        address /*position*/,
        address dst,
        uint256 amount
    ) internal override returns (uint256) {
        // @audit-note: This is still a bug.
@>      uint256 collateralWithdrawn = ICDPVault(vault).withdraw(address(this), amount);

        // if collateral is not the dst token, we need to withdraw the underlying from the ERC4626 vault
        address collateral = address(ICDPVault(vault).token());
        if (dst != collateral) {
            collateralWithdrawn = IERC4626(collateral).redeem(collateralWithdrawn, address(this), address(this));
        }

        return collateralWithdrawn;
    }
```

### Recommended Mitigation Steps

```diff
-       uint256 collateralWithdrawn = ICDPVault(vault).withdraw(address(this), amount);
+       uint256 collateralWithdrawn = ICDPVault(vault).withdraw(position, amount);
```

**[amarcu (LoopFi) confirmed](https://github.com/code-423n4/2024-10-loopfi-findings/issues/13#event-14870931535)**

***

---
