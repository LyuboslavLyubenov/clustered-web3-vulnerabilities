# Cluster -1287

**Rank:** #383  
**Count:** 14  

## Label
Inconsistent or incorrect fee calculation due to improper rounding, missing logic paths, or flawed input handling, leading to revenue loss, free swaps, or denial-of-service via invalid transaction validation.

## Cluster Information
- **Total Findings:** 14

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

**Auto Label:** Inconsistent or incorrect fee calculation due to improper rounding, missing logic paths, or flawed input handling, leading to revenue loss, free swaps, or denial-of-service via invalid transaction validation.  

**Original Text Preview:**

Severity: Low Risk
Context: Vault.sol#L382-L428
Description: The swap fees are applied in a way such that the effective fee is not equivalent for both swap kinds
EXACT_IN and EXACT_OUT .
• Compute Out Given Exact In ( computeOutGivenExactIn ):
Currently, when the exact token input amounts a1 are specified (SwapKind.EXACT_IN ), the aggregate swap
fee percentage 
 s is applied to the token output amount a0 in order to arrive at the net amount a0net .
// amountOut. Round up to avoid losses during precision loss.
locals.swapFeeAmountScaled18 = amountCalculatedScaled18.mulUp(swapState.swapFeePercentage);
// Need to update ` amountCalculatedScaled18 ` for the onAfterSwap hook.
amountCalculatedScaled18 -= locals.swapFeeAmountScaled18;
This translates to the following formula.
a0net = computeOutGivenExactIn(a1)  (1 � 
 s)
= a0  ¯
 s ,
where ¯
 s = 1 � 
 s.
• Compute In Given Exact Out ( computeInGivenExactOut ):
In case the exact token output amount a0 is specified (SwapKind.EXACT_OUT ) the fees are applied as follows.
locals.swapFeeAmountScaled18 = amountCalculatedScaled18.mulDivUp(
swapState.swapFeePercentage,
swapState.swapFeePercentage.complement()
);
amountCalculatedScaled18 += locals.swapFeeAmountScaled18;
a1net = computeInGivenExactOut(a0)  (1 + 
 s
1 � 
 s
)
a1net = computeInGivenExactOut(a0)  ( 1
1 � 
 s
)
a1net = computeInGivenExactOut(a0)  ( 1
¯
 s
)
However, when starting out with the formula a0net = a0  ¯
 s, it is not possible to derive a closed form solution which
applies a constant factor to the output of the computations to both cases, e.g. a1net = a1  ˆ
 s, as ˆ
 s will depend on
the current balances, the output amount and the invariant formula.
If we start by assuming that we want to apply the swap fees to the output when given the exact input amount
a0net = computeOutGivenExactIn(a1)  ¯
 s, we can derive the following formulas (exemplified using the input/output
formulas given the weighted pool's swap invariant).
a0 = b0
"
1 �
 b1
b1 + a1
 w1
w0
## a1 = b1
"  b0
b0 � a0
 w0
w1
� 1
## In order to apply the fee to the output amount, we can make the following substitution for the formulas: a0 = a0
¯
 s
.
=) a0
0 = ¯
 sb0
"
1 �
 b1
b1 + a1
 w1
w0
## = computeOutGivenExactIn(a1)  ¯
 s
=) a1 = b1
@ b0
b0 �
a0
¯
 s
A
w0
w1
� 1
= computeInGivenExactOut(a0
¯
 s
)
In these new formulas, a0
0 can be seen as the net output amount a0net . In order to apply the fees equivalently, the
fees need to scale the output amount first, before applying the formula which computes the output computeIn-
GivenExactOut .
In general, it is not possible to derive a formula applying a constant factor to the result for both cases.
Impact: Low, the actual impact and difference in fees is quite low. This does not affect protocol solvency. In certain
cases, the actual fees charged might differ when specifying EXACT_IN versus EXACT_OUT . One might lead to less
or more fees applied (dependent on the pool's liquidity and input amount).
Likelihood: High, this case is reached whenever the exact token output amount is specified.
Recommendation: Adjust the formulas such that the fee is applied correctly by scaling the given output value by
¯
 s
before computing the required token input amount.
a0net = computeInGivenExactOut(a0
¯
 s
)
Balancer: Fixed in PR 983.
Cantina Managed: Fixed.

---
