# Cluster -1294

**Rank:** #211  
**Count:** 52  

## Label
Truncation in integer arithmetic causes cumulative precision loss, enabling attackers to evade fees, manipulate debt distribution, and drain funds through systematic rounding errors in interest and share calculations.

## Cluster Information
- **Total Findings:** 52

## Examples

### Example 1

**Auto Label:** Precision loss in fixed-point arithmetic due to improper division ordering and truncation, leading to fee underpayment, rounding errors, and exploitable revenue loss.  

**Original Text Preview:**

## Severity: Low Risk

## Context
`PriceAndFeeCalculator.sol#L342-L354`

## Description
The `_convertUnitsToToken()` function in `PriceAndFeeCalculator` converts a given number of units to the amount of a given token. Note that the result is underestimated since the integer division in the calculation rounds down. This function is indirectly used in the two functions: `_unitsToTokensFloor()` and `_unitsToTokensCeil()`:

1. The `_unitsToTokensFloor()` function calculates how many tokens users can receive when redeeming from the vault. Therefore, rounding the result down is correct.
2. The `_unitsToTokensCeil()` function calculates how many tokens users need to provide when depositing into the vault. Therefore, it should use a round-up result in order not to favor the users.

Although the `_unitsToTokensCeil()` rounds up the division when applying the multiplier, since it can be 1, the overall result could still be rounded down. Users could end up paying fewer tokens than they should.

## Recommendation
Consider passing a variable to the `_convertUnitsToToken()` function to signal if the division should round up or down. Round up the division when the function is used in `_unitsToTokensCeil()`.

## Aera
Fixed in PR 342.

## Spearbit
Verified.

---
### Example 2

**Auto Label:** Precision loss in fixed-point arithmetic due to improper division ordering and truncation, leading to fee underpayment, rounding errors, and exploitable revenue loss.  

**Original Text Preview:**

##### Description
This issue has been identified within the `addLiquidity` function of the `LPManager` contract. 

Integer division in fee calculations can allow an attacker to exploit rounding, shaving off small portions of fees. Repeated usage could slightly diminish the overall fees collected.

The issue is classified as **low** severity because the financial impact is minimal, though it may accumulate over time.

##### Recommendation
We recommend using rounding-up division (e.g. using `solmate.FixedPointMathLib.mulDivUp`) in fee calculations to ensure no fractions are lost due to division.

---

---
### Example 3

**Auto Label:** Precision loss in fixed-point arithmetic due to improper division ordering and truncation, leading to fee underpayment, rounding errors, and exploitable revenue loss.  

**Original Text Preview:**

##### Description
This issue has been identified within the `addLiquidity` function of the `LPManager` contract. 

Integer division in fee calculations can allow an attacker to exploit rounding, shaving off small portions of fees. Repeated usage could slightly diminish the overall fees collected.

The issue is classified as **low** severity because the financial impact is minimal, though it may accumulate over time.

##### Recommendation
We recommend using rounding-up division (e.g. using `solmate.FixedPointMathLib.mulDivUp`) in fee calculations to ensure no fractions are lost due to division.

---

---
