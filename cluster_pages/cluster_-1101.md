# Cluster -1101

**Rank:** #54  
**Count:** 313  

## Label
Failure to validate array length equality during iteration leads to out-of-bounds access, incorrect state updates, or infinite loops, enabling denial-of-service, data corruption, or unintended transactions.

## Cluster Information
- **Total Findings:** 313

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

**Auto Label:** Failure to validate array length equality during iteration leads to out-of-bounds access, incorrect state updates, or infinite loops, enabling denial-of-service, data corruption, or unintended transactions.  

**Original Text Preview:**

In the [`_executeCall` function](https://github.com/matter-labs/zksync-sso-clave-contracts/blob/c7714c0fe0a33a23acce5aa20355f088d330b4f7/src/SsoAccount.sol#L128) of the `SsoAccount` contract, there is a slice operation performed on the `_data` parameter when handling a `DEPLOYER_SYSTEM_CONTRACT` call. The purpose of this slice operation is to retrieve the function selector, which requires at least 4 bytes of data. However, the current implementation does not validate whether the input data has sufficient length. If the provided data is shorter than 4 bytes, the slicing operation will cause a panic, abruptly terminating the execution.

Consider verifying the length of the `_data` parameter prior to slicing, and explicitly reverting with a meaningful error message if the length is insufficient.

***Update:** Resolved in [pull request #369](https://github.com/matter-labs/zksync-sso-clave-contracts/pull/369) at commit [f476fb1](https://github.com/matter-labs/zksync-sso-clave-contracts/pull/369/commits/f476fb1f064a180ddfcb906c8468dac98c99da1d).*

---
