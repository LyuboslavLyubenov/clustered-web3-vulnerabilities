# Cluster -1299

**Rank:** #228  
**Count:** 45  

## Label
Flawed mathematical calculations in fee and share accrual lead to inaccurate distributions, inflated fees, reduced LP returns, and systemic underestimation of fee floors due to imprecise fixed-point arithmetic.

## Cluster Information
- **Total Findings:** 45

## Examples

### Example 1

**Auto Label:** Flawed mathematical calculations in fee and share accrual lead to inaccurate distributions, inflated fees, reduced LP returns, and systemic underestimation of fee floors due to imprecise fixed-point arithmetic.  

**Original Text Preview:**

## Diﬃculty: High

## Type: Denial of Service

## Description
The contract converts the annual percentage fee floor to a per-second rate using an approximation that introduces precision loss. The issue stems from the `ONE_OVER_YEAR` constant, which represents `1/31536000` (seconds in a standard 365-day year) and is used in the conversion formula:

```solidity
uint256 feeFloor = D18 - MathLib.pow(D18 - daoFeeFloor, ONE_OVER_YEAR);
```

**Figure 4.1:** feeFloor calculation (reserve-index-dtf/contracts/Folio.sol#937)

This constant is inherently imprecise due to fixed-point arithmetic limitations; here it is rounded down. As a result, the actual enforced fee floor is slightly lower than intended. For example, a configured annual fee floor of 0.15% becomes approximately 0.14999990833183% after conversion.

Additionally, this calculation does not account for leap years, which have 366 days or 31,622,400 seconds instead of 31,536,000 seconds. During leap years, the `ONE_OVER_YEAR` constant is even more inaccurate, as it still uses the number of seconds in a standard year, causing the calculated per-second fee floor to be further off from the expected value. This means that the DAO will receive a higher percentage of fees during leap years compared to regular years for the same `feeFloor` stored variable on the `FolioDAOFeeRegistry` that represents the yearly fee floor.

## Recommendations
- **Short term:** Document this precision loss in both code comments and user-facing documentation to set appropriate expectations for the DAO regarding minimum fees and leap years. Consider adjusting the fee floor value slightly upward to account for the known precision loss if needed.
- **Long term:** Consider adding comprehensive documentation for every instance of precision loss throughout the protocol to ensure all users interacting with the system are aware of these limitations.

---
### Example 2

**Auto Label:** Rounding errors in fee calculations due to improper integer arithmetic lead to systematic underpayment of protocol revenue, eroding economic integrity and incentivizing liquidity withdrawal.  

**Original Text Preview:**

**Description:** Protocol fee should round up in favor of the protocol in `onMorphoFlashLoan`:
```solidity
uint256 rockoFeeBP = ROCKO_FEE_BP;
if (rockoFeeBP > 0) {
    unchecked {
        feeAmount = (flashBorrowAmount * rockoFeeBP) / BASIS_POINTS_DIVISOR;
        borrowAmountWithFee += feeAmount;
    }
}
```

Consider [using](https://x.com/DevDacian/status/1892529633104396479) OpenZeppelin [`Math::mulDiv`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/fda6b85f2c65d146b86d513a604554d15abd6679/contracts/utils/math/Math.sol#L280) with the rounding parameter or Solady [`FixedPointMathLib::fullMulDivUp`](https://github.com/Vectorized/solady/blob/c9e079c0ca836dcc52777a1fa7227ef28e3537b3/src/utils/FixedPointMathLib.sol#L548).

Another benefit of using these libraries is that intermediate overflow from the multiplication of `flashBorrowAmount * rockoFeeBP` is avoided.

**Rocko:** Fixed in commit [a59ba0e](https://github.com/getrocko/onchain/commit/a59ba0e7958c544ad95788ce29923a342a2ea35a).

**Cyfrin:** Verified.

---
### Example 3

**Auto Label:** Systematic fee under-collection due to flawed arithmetic and improper fee modeling, leading to significant financial losses and compromised liquidity accuracy.  

**Original Text Preview:**

**Impact**

`getSwapFee` computes the fee to be paid as follows:

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/SwapPair.sol#L196-L199

```solidity
      if ( /// @audit No deviation threshold | Spot vs Oracle
        (postDexPrice > oraclePrice && postDexPrice > preDexPrice) || /// @audit Logical mechanism of swapping?
        (postDexPrice < oraclePrice && preDexPrice > postDexPrice)
      ) {
```

We can chart out the logic as follows:

- Post  > Oracle
- Post > Prev > Oracle
- Post > Oracle > Prev

- Post < Oracle
- Post < Prev < Oracle 
- Post < Oracle < Prev

Meaning this is charging a fee whenever the absolute postDexPrice surpasses the absolute oracle price

So when crossing the impact is "halved"

When not crossing the middle price, the average between the two dex prices will be very distant from the oracle price, causing the fee to be higher, whereas when crossing the fee would effectively be halved

**Mitigation**

I don't believe there's any need for a specific mitigation

---
