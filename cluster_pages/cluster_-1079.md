# Cluster -1079

**Rank:** #131  
**Count:** 107  

## Label
Flawed input validation and state variable misuse lead to incorrect economic assumptions, misconfigurations, and miscalculated financial settlements, compromising contract integrity and potentially causing financial loss or system failure.

## Cluster Information
- **Total Findings:** 107

## Examples

### Example 1

**Auto Label:** Failure to validate inputs or maintain consistent state leads to incorrect asset handling, fund loss, or erroneous accounting due to unverified assumptions, inconsistent indexing, or redundant state variables.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Low

## Description

The `updateBitFlag` function allows the basket's asset list to be updated, with the restriction that the new `bitFlag` must be a superset of the previous `bitFlag`. During the `updateBitFlag` process, if all validations pass, the function iterates over the new asset list and updates `basketAssetToIndexPlusOne`. However, it does not update `basketTokenToBaseAssetIndexPlusOne`.

Updating a `bitFlag` that is a superset of the previous `bitFlag` can still potentially impact `basketTokenToBaseAssetIndexPlusOne`. Consider the following scenario:

1. The current bitFlag is `1110`, meaning the asset list contains three assets. In this scenario, the first asset in the returned array is set as `basketTokenToBaseAssetIndexPlusOne`, resulting in a value of 1 (0 + 1).
2. The new bitFlag is `1111`, which is a superset of the previous `bitFlag`. The asset list now contains four assets. However, since `basketTokenToBaseAssetIndexPlusOne` has not been updated, the base asset index will point to the newly included asset. This causes all functions relying on this index to process the wrong asset.

## Recommendations

Consider updating `basketTokenToBaseAssetIndexPlusOne` when `updateBitFlag` is called.

---
### Example 2

**Auto Label:** Failure to properly validate or manage state transitions leads to persistent misconfigurations, incorrect asset pricing, funding leaks, and flawed callback flows, undermining financial integrity and operational reliability.  

**Original Text Preview:**

The `configToken` function allows admins to whitelist a token and set its price feed.

```solidity
    function configToken(address token, address priceFeed) external onlyRole(AccessControlStorage.DEFAULT_ADMIN_ROLE) {
        if (address(tokenOracle[token]) != address(0)) revert TokenAlreadyConfigured();
        else whitelistedTokens.push(token);

        tokenOracle[token] = AggregatorV3Interface(priceFeed);
        emit TokenAdded(token);
    }
```

However, once a token is configured, it cannot be updated or removed. This becomes problematic if the token or its associated price feed becomes faulty, deprecated, or incorrect.

There is no way to recover from misconfiguration or respond to issues with oracles, which introduces operational risk and limits the flexibility of the system.

---
### Example 3

**Auto Label:** Inadequate input and state validation leads to inconsistent, exploitable system configurations and data integrity failures, enabling unauthorized state changes and financial manipulation.  

**Original Text Preview:**

In cases where the edge between tokens is not set, the `edge()` function will return a default edge. However, this default edge is not properly checked for correctness. Specifically, when `defaultEdge.amplitude == 0`, the `edge()` function should `revert` to prevent using an invalid edge, as it is used in the `SwapFacet._preSwap()` and `LiqFacet.addLiq()` functions this could lead to unexpected behavior.

The issue arises from the removal of the proper check in the `Edge.getSlot0()` function, which was never reintroduced in the `Store::edge()` function. Additionally, the removal of this check leaves the `error NoEdgeSettings(address token0, address token1);` unused in the code.

Consider adding the missing sanity check to the `Store::edge()` function.

---
