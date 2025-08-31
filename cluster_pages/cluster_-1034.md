# Cluster -1034

**Rank:** #202  
**Count:** 54  

## Label
Inaccurate calculations due to improper division, zero checks, or flawed precision handling lead to incorrect pricing, share valuation, or liquidity management, enabling financial misrepresentation, deadlocks, or exploitable edge-case manipulations.

## Cluster Information
- **Total Findings:** 54

## Examples

### Example 1

**Auto Label:** Improper arithmetic and state mutation lead to incorrect allocations, dust loss, or division-by-zero, compromising financial accuracy and transaction reliability.  

**Original Text Preview:**

## Security Report

## Severity
**Low Risk**

## Context
LockingController.sol#L75

## Description
There is no validation on the `_unwindingEpochs` argument of `LockingController.enableBucket`. This value is later used in `UnwindingModule.startUnwinding` as the denominator for a calculation:

```solidity
uint256 rewardWeightDecrease = totalDecrease / uint256(_unwindingEpochs);
```

If GOVERNOR accidentally enabled a bucket with 0 unwinding epochs, unwinding would revert.

## Recommendation
Enforce a reasonable range to prevent GOVERNOR footguns.

## Status
- **infiniFi:** Fixed in 89d123a.
- **Spearbit:** Fix verified.

---
### Example 2

**Auto Label:** Improper arithmetic and state mutation lead to incorrect allocations, dust loss, or division-by-zero, compromising financial accuracy and transaction reliability.  

**Original Text Preview:**

## Audit Findings

## Severity
**Low Risk**

## Context
LockingController.sol#L256-L257

## Description
The `increaseUnwindingEpochs()` function rebalances shares from the old bucket to the new bucket. The `receiptTokens` calculation uses `oldShares`, `totalReceiptTokens`, and `oldTotalSupply` in the line:

```solidity
uint256 receiptTokens = oldShares.mulDivDown(oldData.totalReceiptTokens, oldTotalSupply);
```

When a user has a small amount of shares (`oldShares`), this calculation can round down to zero, causing the function to return early. As a result, any tokens transferred to the gateway become lost.

## Recommendation
Consider returning any dust amount of shares to the user at the end of execution by checking the gateway balance and transferring the remainder back to the user.

## Acknowledgments
- **infiniFi:** Acknowledged, won't fix.
- **Spearbit:** Acknowledged.

---
### Example 3

**Auto Label:** Improper arithmetic and state mutation lead to incorrect allocations, dust loss, or division-by-zero, compromising financial accuracy and transaction reliability.  

**Original Text Preview:**

## Vulnerability Report

## Severity
**High Risk**

## Context
**File:** LockingController.sol  
**Location:** Line 415

## Description
In the `applyLosses` function, the code attempts to distribute the negative `yield_amount` among each epoch bucket by computing:

```solidity
uint256 allocation = epochTotalReceiptToken.mulDivUp(_amount, _globalReceiptToken);
_globalReceiptToken -= allocation;
```

Because `_globalReceiptToken` is decremented in each iteration of the for loop, each bucket's allocation is calculated over a progressively smaller denominator. This will cause the sum of all allocations to exceed `_amount`, leading to more slash than intended. Worse, at the end of the loop, `globalReceiptToken` and `globalRewardWeight` are incorrectly set to these artificially decremented values, corrupting the final accounting.

## Recommendation
To fix this accounting error and ensure accurate loss distribution, the allocation calculation should use the state variable `globalReceiptToken` as the denominator instead of `_globalReceiptToken`. The corrected line of code should read:

```solidity
uint256 allocation = epochTotalReceiptToken.mulDivUp(_amount, globalReceiptToken);
```

## Status
**infiniFi:** Fixed in b9af8d.  
**Spearbit:** Fix verified.

---
