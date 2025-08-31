# Cluster -1064

**Rank:** #264  
**Count:** 32  

## Label
Out-of-bounds array access leading to undefined behavior, incorrect selections, or denial-of-service due to improper index validation and unbounded iteration.

## Cluster Information
- **Total Findings:** 32

## Examples

### Example 1

**Auto Label:** Out-of-bounds array access leading to undefined behavior, incorrect selections, or denial-of-service due to improper index validation and unbounded iteration.  

**Original Text Preview:**

Inside `updateForRange`, the provided `end` value is used to loop over the `pools` without checking whether it exceeds the valid `pools` range`.

```solidity
    function updateForRange(uint start, uint end) public {
        for (uint i = start; i < end; i++) {
            _updateFor(gauges[pools[i]]);
        }
    }
```

This could result in a revert. Consider checking the `end` value, and if it exceeds the length of pools, use the pools length instead.

```diff
    function updateForRange(uint start, uint end) public {
+         end = Math.min(end, pools.length);
        for (uint i = start; i < end; i++) {
            _updateFor(gauges[pools[i]]);
        }
    }
```

---
### Example 2

**Auto Label:** Out-of-bounds array access leading to undefined behavior, incorrect selections, or denial-of-service due to improper index validation and unbounded iteration.  

**Original Text Preview:**

Inside `updateForRange`, the provided `end` value is used to loop over the `pools` without checking whether it exceeds the valid `pools` range`.

```solidity
    function updateForRange(uint start, uint end) public {
        for (uint i = start; i < end; i++) {
            _updateFor(gauges[pools[i]]);
        }
    }
```

This could result in a revert. Consider checking the `end` value, and if it exceeds the length of pools, use the pools length instead.


```diff
    function updateForRange(uint start, uint end) public {
+         end = Math.min(end, pools.length);
        for (uint i = start; i < end; i++) {
            _updateFor(gauges[pools[i]]);
        }
    }
```

---
### Example 3

**Auto Label:** Out-of-bounds array access leading to undefined behavior, incorrect selections, or denial-of-service due to improper index validation and unbounded iteration.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

1. The `totalAssets()` function in `OmoVault.sol` contains an unbounded loop that iterates through all registered accounts:
2. `OmoRouter.registerAccount` loops through all accounts, making it susceptible to a DoS attack via createAccount().
3. `OmoVaulFactory.getVaultId` loops through all vaults...
4. `OmoAgent.topOffToAgent` has unbounded loop when tries to remove `positionId`
5. `OmoAgent._removeTokenAddress` loops through all token addresses
6. `AgentSetter.removeAgent` loops through all `whitelistedAgents` which is unbounded

## Recommendation

Consider adding maximum limits or use mappings instead of arrays for storage

---
