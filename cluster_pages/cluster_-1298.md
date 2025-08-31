# Cluster -1298

**Rank:** #266  
**Count:** 32  

## Label
Redundant computations and inefficient storage layouts lead to increased gas costs and reduced performance through repeated calculations, poor data type selection, and suboptimal state access.

## Cluster Information
- **Total Findings:** 32

## Examples

### Example 1

**Auto Label:** Redundant computations and inefficient storage layouts lead to increased gas costs and reduced performance through repeated calculations, poor data type selection, and suboptimal state access.  

**Original Text Preview:**

##### Description

The `modifyPosition` function calls `getPerpetualPriceForPosition` multiple times with the same input parameters, leading to redundant computations. These repeated calls increase gas costs and reduce efficiency.

Since the parameters remain unchanged during execution, the same result is recalculated unnecessarily instead of being reused.

##### BVSS

[AO:S/AC:L/AX:L/C:N/I:L/A:L/D:N/Y:N/R:F/S:C (0.2)](/bvss?q=AO:S/AC:L/AX:L/C:N/I:L/A:L/D:N/Y:N/R:F/S:C)

##### Recommendation

Store the result of `getPerpetualPriceForPosition` in a temporary variable and reuse it throughout the function. For example:

```
uint256 perpetualPrice = getPerpetualPriceForPosition(key, indexPrice, int256(size), isLong);

```

Ensure that no parameter changes occur during execution that could affect the result. This optimization reduces gas consumption and improves performance without altering the logic.

##### Remediation

**ACKNOWLEDGED:** The **Dexodus team** acknowledged this finding.

---
### Example 2

**Auto Label:** Redundant storage reads and writes leading to unnecessary gas costs and inefficient state mutations.  

**Original Text Preview:**

**Severity** - Informational

**Status** - Resolved

**Location** - StakingERC1155Contract.sol

**Description**

The withdraw() and slash() functions read _s.amount from storage once more for the zero comparison even if the _amount memory variable is declared.

**Recommendation** 

Use _amount instead of _s.amount for the zero comparison.

---
### Example 3

**Auto Label:** Redundant computation due to repeated function or field access, leading to unnecessary gas costs and performance inefficiencies.  

**Original Text Preview:**

## Code Review Summary

## Context
- **File:** MainnetController.sol
- **Lines:** #L246, #L276

## Description
The `psm.to18ConversionFactor()` call can be cached as this variable is immutable in the psm.

## Recommendation
Consider caching this variable into an immutable variable.

## MakerDAO
Fixed in commit `5ea7f6d2`.

## Cantina Managed
Verified.

---
