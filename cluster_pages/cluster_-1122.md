# Cluster -1122

**Rank:** #225  
**Count:** 46  

## Label
Insufficient input validation and improper state transitions enable attackers to manipulate fees, exploit timing, or bypass safeguards, leading to unauthorized revenue capture, overcharging, or denial of service.

## Cluster Information
- **Total Findings:** 46

## Examples

### Example 1

**Auto Label:** Inconsistent or incorrect fee calculation due to improper rounding, missing logic paths, or flawed input handling, leading to revenue loss, free swaps, or denial-of-service via invalid transaction validation.  

**Original Text Preview:**

## Severity

Medium

## Description

The settleMatchedOrders function allows the backend to settle matched orders that
were created and matched off-chain. While the contract enforces an upper bound for feeRate, it does
not enforce a minimum feerate.
This omission allows orders with extremely low feeRate values(e.g.,1wei)to be matched and settled.
As aresult, users can effectively bypass protocol fees altogether, undermining protocol revenue and
fairness.
Since order matching is handled off-chain and executed on-chain by an address with the BACK
END_ROLE, malicious or careless matching could lead to widespread abuse of low-fee orders.

## Recommendation

Introduce a minFeeRate variable, configurable by protocol administrators, and
enforce it during order validation to ensure all orders pay at least a minimum protocol fee:

```solidity
require(order.feeRate >= minFeeRate, "Fee rate below minimum allowed");
```

## Team Response

Acknowledged

---
### Example 2

**Auto Label:** Inconsistent or incorrect fee calculation due to improper rounding, missing logic paths, or flawed input handling, leading to revenue loss, free swaps, or denial-of-service via invalid transaction validation.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-04-burve-judging/issues/311 

## Found by 
0L5S6W, 0x23r0, AshishLac, CodexBugmeNot, Cybrid, Egbe, JeRRy0422, Rhaydden, SecSnat, Sparrow\_Jac, TheCarrot, Tigerfrake, VinciGearHead, aman, anirruth\_, baz1ka, benjamin\_0923, bladeee, bretzel, curly11, elolpuer, future, gh0xt, heeze, holydevoti0n, imageAfrika, kom, m3dython, prosper, rsam\_eth, smartkelvin, v1c7, vangrim

### Summary

In the `ValueFacet.removeValueSingle` function, the local variable `removedBalance` is read (as `0`) when computing the real fee (`realTax`), allowing users to withdraw single‐token removals without ever paying the protocol fee. This results in a **complete bypass of the intended swap/remove fee**, leading to potential revenue loss.


### Root Cause

The function’s return variable `removedBalance` is declared but not yet assigned when used to compute `realTax`. The intent was to prorate the nominal fee (`nominalTax`) to “real” token units by computing  


```solidity
function removeValueSingle(
    address recipient,
    uint16 _closureId,
    uint128 value,
    uint128 bgtValue,
    address token,
    uint128 minReceive
) external nonReentrant returns (uint256 removedBalance) {
    …
    (uint256 removedNominal, uint256 nominalTax) = c.removeValueSingle(
        value,
        bgtValue,
        vid
    );
    uint256 realRemoved = AdjustorLib.toReal(token, removedNominal, false);
    Store.vertex(vid).withdraw(cid, realRemoved, false);
    // BUG: removedBalance is still zero here
    uint256 realTax = FullMath.mulDiv(
        removedBalance,        // ← should be realRemoved
        nominalTax,
        removedNominal
    );
    c.addEarnings(vid, realTax);
    removedBalance = realRemoved - realTax;
    require(removedBalance >= minReceive, PastSlippageBounds());
    TransferHelper.safeTransfer(token, recipient, removedBalance);
}
```
https://github.com/sherlock-audit/2025-04-burve/blob/main/Burve/src/multi/facets/ValueFacet.sol#L214-L245


```solidity
realTax = realRemoved * nominalTax  / removedNominal;
```
but instead `removedBalance` (zero) is used as the numerator, making `realTax == 0` always.




### Internal Pre-conditions

N/A

### External Pre-conditions

N/A

### Attack Path

N/A

### Impact

The protocol loses **100% of the intended fees** on every single‐token removal.


### PoC

_No response_

### Mitigation


Replace the incorrect use of `removedBalance` with `realRemoved` when computing `realTax`. Specifically:

```diff
-    uint256 realTax = FullMath.mulDiv(
-        removedBalance,
-        nominalTax,
-        removedNominal
-    );
+    // Compute the real‐world fee based on the actual tokens removed
+    uint256 realTax = FullMath.mulDiv(
+        realRemoved,
+        nominalTax,
+        removedNominal
+    );
```

## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/itos-finance/Burve/commit/ca58c88be97e8f3f7a4617b171377d77fca52cdc

---
### Example 3

**Auto Label:** Timing-based attacks exploiting missing access controls, validation checks, or state transitions enable unauthorized fee manipulation, front-running, and fund redirection.  

**Original Text Preview:**

**Impact**

Major changes in V2 include:
- 72% of interest raid paid out to SP stakers
- Inability to borrow new debt in RM


The logic can be seen in `_requireValidAdjustmentInCurrentMode\
https://github.com/liquity/bold/blob/a34960222df5061fa7c0213df5d20626adf3ecc4/contracts/src/BorrowerOperations.sol#L1359-L1386

```solidity
    function _requireValidAdjustmentInCurrentMode(
        TroveChange memory _troveChange,
        LocalVariables_adjustTrove memory _vars
    ) internal view {
        /*
        * Below Critical Threshold, it is not permitted:
        *
        * - Borrowing
        * - Collateral withdrawal except accompanied by a debt repayment of at least the same value
        *
        * In Normal Mode, ensure:
        *
        * - The adjustment won't pull the TCR below CCR
        *
        * In Both cases:
        * - The new ICR is above MCR
        */
        _requireICRisAboveMCR(_vars.newICR);

        if (_vars.isBelowCriticalThreshold) {
            _requireNoBorrowing(_troveChange.debtIncrease);
            _requireDebtRepaymentGeCollWithdrawal(_troveChange, _vars.price);
        } else {
            // if Normal Mode
            uint256 newTCR = _getNewTCRFromTroveChange(_troveChange, _vars.price);
            _requireNewTCRisAboveCCR(newTCR);
        }
    }
```

This opens up to a DOS by performing the following:
- Borrow as close as possible to CCR
- The interest rate of the next block will trigger RM, disallowing new borrows

This can be weaponized in the 2 following scenarios:

**Whale shuts down the branch**
1) A Whale sees that another SP is providing more yield, to have no competition, they borrow at a cheap rate, and disable borrowing from the branch via the above
2) They stake in the other SP, and receive a higher interest than what they pay

**Initial Depositor disables the branch**
At the earliest stages of the system, triggering CCR will be very inexpensive, as it will require very little debt and collateral

This could possibly performed on all branches simultaneosly, with the goal of making Liquity V2 unable to be bootstrapped


**Current Mitigation**

Both scenarios are somewhat mitigated by Redemptions, with one key factor to keep in mind: the unprofitability to the redeemer

Redemptions charge a fee
Based on oracle drift, the collateral redeemed may be worth more or less than what the oracle reports

Because of these factors, redemptions will be able to disarm the grief and restore TCR > CCR (in most cases), but this operation may be unprofitable to the redeemer whom will have to perform it just to be able to use the system


**Mitigation**

I believe opening troves should be allowed during CCR, as long as they raise the TCR 

Alternatively you could discuss with partners a strategy to seed BOLD with enough collateral to prevent any reasonable way to grief it, this technically will cost the partners their cost of capital

---
