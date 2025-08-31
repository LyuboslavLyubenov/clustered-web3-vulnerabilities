# Cluster -1275

**Rank:** #137  
**Count:** 104  

## Label
Hardcoded numerical values lack context, increase error risk, and undermine readability and auditability in critical logic paths, leading to maintainability and security vulnerabilities.

## Cluster Information
- **Total Findings:** 104

## Examples

### Example 1

**Auto Label:** Hardcoded values and inconsistent constant usage lead to reduced readability, increased error risk, and compromised maintainability through ambiguous or mutable logic.  

**Original Text Preview:**

##### Description
Although the `LimitOrderManager` contract defines `bytes internal constant ZERO_BYTES = bytes('');`, three calls to `IAlgebraPool.burn()` still pass the literal empty string `''` instead of the constant, reducing consistency and readability.
<br/>
##### Recommendation
We recommend replacing the string literals with `ZERO_BYTES` in all relevant calls.

> **Client's Commentary:**
> Commit: https://github.com/cryptoalgebra/plugins-monorepo/commit/ddcf5daace8d74784eb962e3bde0c16e2dcef90e

---

---
### Example 2

**Auto Label:** Hardcoded numerical values lack context, increase error risk, and undermine readability and auditability in critical logic paths, leading to maintainability and security vulnerabilities.  

**Original Text Preview:**

##### Description
The `DIAWhitelistedStaking` contract uses the hardcoded value `24 * 60 * 60` to calculate `passedDays` in its reward logic. However, the constant `SECONDS_PER_DAY` is already defined in the `StakingErrorsAndEvents` file for this exact purpose. Using the hardcoded expression instead of the named constant introduces inconsistency in the codebase and reduces readability.
<br/>
##### Recommendation
We recommend replacing the hardcoded `24 * 60 * 60` expression in the `passedDays` calculation with the `SECONDS_PER_DAY` constant defined in `StakingErrorsAndEvents`. This promotes code consistency and improves maintainability across the staking contracts.

> **Client's Commentary:**
> hardcoded values are moved to use constant

---
### Example 3

**Auto Label:** Hardcoded numerical values lack context, increase error risk, and undermine readability and auditability in critical logic paths, leading to maintainability and security vulnerabilities.  

**Original Text Preview:**

**Description:** `LibDoubleGeometricDistribution` and `LibCarpetedDoubleGeometricDistribution` both define the following constant:

```solidity
uint256 internal constant MIN_LIQUIDITY_DENSITY = Q96 / 1e3;
```

However, both instances are unused and can be removed as they appear to have been erroneously copied from the `LibGeometricDistribution` and `LibCarpetedGeometricDistribution` libraries.

**Bacon Labs:** Fixed in [PR \#112](https://github.com/timeless-fi/bunni-v2/pull/112).

**Cyfrin:** Verified, the unused constants have been removed from `LibCarpetedDoubleGeometricDistribution`.

---
