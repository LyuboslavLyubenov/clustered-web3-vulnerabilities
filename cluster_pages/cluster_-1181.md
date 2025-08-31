# Cluster -1181

**Rank:** #145  
**Count:** 96  

## Label
Inconsistent or incorrect fee calculation due to improper rounding, missing logic paths, or flawed input handling, leading to revenue loss, free swaps, or denial-of-service via invalid transaction validation.

## Cluster Information
- **Total Findings:** 96

## Examples

### Example 1

**Auto Label:** Failure to properly persist or validate fees in storage or logic, resulting in irreversible user fund loss, incorrect fee accumulation, or ineffective fee control.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

Inside `CLGauge.claimFees`, it will call `pool.collectFees` and if returned `claimed0` and `claimed1` is greater than 0, it will consider it as fees.

```solidity
    function _claimFees() internal returns (uint claimed0, uint claimed1) {
        if (!isForPair) return (0, 0);
        (claimed0, claimed1) = pool.collectFees();
>>>     if (claimed0 > 0 || claimed1 > 0) {
            uint _fees0 = fees0 + claimed0;
            uint _fees1 = fees1 + claimed1;

            if (
                _fees0 > IBribe(internal_bribe).left(token0) &&
                _fees0 / ProtocolTimeLibrary.WEEK > 0
            ) {
                fees0 = 0;
                _safeApprove(token0, internal_bribe, _fees0);
                IBribe(internal_bribe).notifyRewardAmount(token0, _fees0);
            } else {
                fees0 = _fees0;
            }
            if (
                _fees1 > IBribe(internal_bribe).left(token1) &&
                _fees1 / ProtocolTimeLibrary.WEEK > 0
            ) {
                fees1 = 0;
                _safeApprove(token1, internal_bribe, _fees1);
                IBribe(internal_bribe).notifyRewardAmount(token1, _fees1);
            } else {
                fees1 = _fees1;
            }

            emit ClaimFees(msg.sender, claimed0, claimed1);
        }
    }
```

However, inside `pool`, it will set fees amount to 1 if it is empty.

```solidity
    function collectFees()
        external
        override
        lock
        onlyGauge
        returns (uint128 amount0, uint128 amount1)
    {
        amount0 = gaugeFees.token0;
        amount1 = gaugeFees.token1;
        if (amount0 > 1) {
            // ensure that the slot is not cleared, for gas savings
>>>         gaugeFees.token0 = 1;
            TransferHelper.safeTransfer(token0, msg.sender, --amount0);
        }
        if (amount1 > 1) {
            // ensure that the slot is not cleared, for gas savings
>>>         gaugeFees.token1 = 1;
            TransferHelper.safeTransfer(token1, msg.sender, --amount1);
        }

        emit CollectFees(msg.sender, amount0, amount1);
    }
```

This means if currently fees are empty, `CLGauge` will incorrectly add 1 fee to the `fees0` and `fees1`, which will eventually a cause revert due to lack of balance.

## Recommendations

Adjust the following line inside `CLPool`.

```diff
    function collectFees() external override lock onlyGauge returns (uint128 amount0, uint128 amount1) {
        amount0 = gaugeFees.token0;
        amount1 = gaugeFees.token1;
        if (amount0 > 1) {
            // ensure that the slot is not cleared, for gas savings
            gaugeFees.token0 = 1;
            TransferHelper.safeTransfer(token0, msg.sender, --amount0);
-        }
+        } else {
+           --amount0;
+      }
        if (amount1 > 1) {
            // ensure that the slot is not cleared, for gas savings
            gaugeFees.token1 = 1;
            TransferHelper.safeTransfer(token1, msg.sender, --amount1);
-        }
+        } else {
+           --amount1;
+      }

        emit CollectFees(msg.sender, amount0, amount1);
    }
```

---
### Example 2

**Auto Label:** Failure to properly persist or validate fees in storage or logic, resulting in irreversible user fund loss, incorrect fee accumulation, or ineffective fee control.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

Inside `CLGauge.claimFees`, it will call `pool.collectFees` and if returned `claimed0` and `claimed1` is greater than 0, it will consider it as fees.

```solidity
    function _claimFees() internal returns (uint claimed0, uint claimed1) {
        if (!isForPair) return (0, 0);
        (claimed0, claimed1) = pool.collectFees();
>>>     if (claimed0 > 0 || claimed1 > 0) {
            uint _fees0 = fees0 + claimed0;
            uint _fees1 = fees1 + claimed1;

            if (
                _fees0 > IBribe(internal_bribe).left(token0) &&
                _fees0 / ProtocolTimeLibrary.WEEK > 0
            ) {
                fees0 = 0;
                _safeApprove(token0, internal_bribe, _fees0);
                IBribe(internal_bribe).notifyRewardAmount(token0, _fees0);
            } else {
                fees0 = _fees0;
            }
            if (
                _fees1 > IBribe(internal_bribe).left(token1) &&
                _fees1 / ProtocolTimeLibrary.WEEK > 0
            ) {
                fees1 = 0;
                _safeApprove(token1, internal_bribe, _fees1);
                IBribe(internal_bribe).notifyRewardAmount(token1, _fees1);
            } else {
                fees1 = _fees1;
            }

            emit ClaimFees(msg.sender, claimed0, claimed1);
        }
    }
```

However, inside `pool`, it will set fees amount to 1 if it is empty.

```solidity
    function collectFees()
        external
        override
        lock
        onlyGauge
        returns (uint128 amount0, uint128 amount1)
    {
        amount0 = gaugeFees.token0;
        amount1 = gaugeFees.token1;
        if (amount0 > 1) {
            // ensure that the slot is not cleared, for gas savings
>>>         gaugeFees.token0 = 1;
            TransferHelper.safeTransfer(token0, msg.sender, --amount0);
        }
        if (amount1 > 1) {
            // ensure that the slot is not cleared, for gas savings
>>>         gaugeFees.token1 = 1;
            TransferHelper.safeTransfer(token1, msg.sender, --amount1);
        }

        emit CollectFees(msg.sender, amount0, amount1);
    }
```

This means if currently fees are empty, `CLGauge` will incorrectly add 1 fee to the `fees0` and `fees1`, which will eventually a cause revert due to lack of balance.

## Recommendations

Adjust the following line inside `CLPool`.

```diff
    function collectFees() external override lock onlyGauge returns (uint128 amount0, uint128 amount1) {
        amount0 = gaugeFees.token0;
        amount1 = gaugeFees.token1;
        if (amount0 > 1) {
            // ensure that the slot is not cleared, for gas savings
            gaugeFees.token0 = 1;
            TransferHelper.safeTransfer(token0, msg.sender, --amount0);
-        } 
+        } else {
+           --amount0;
+      }
        if (amount1 > 1) {
            // ensure that the slot is not cleared, for gas savings
            gaugeFees.token1 = 1;
            TransferHelper.safeTransfer(token1, msg.sender, --amount1);
-        } 
+        } else {
+           --amount1;
+      }

        emit CollectFees(msg.sender, amount0, amount1);
    }
```

---
### Example 3

**Auto Label:** **Inaccurate or missing fee calculations due to flawed cost modeling, improper validation, or lack of real-time on-chain visibility, leading to financial loss, incorrect transactions, or user risk.**  

**Original Text Preview:**

## Severity

Informational

## Description

Currently, relayers calling the withdraw function lack an efficient way to determine the
exact fee they will receive for processing a withdrawal. This uncertainty may discourage participation
or lead to inefficient relaying strategies.

## Recommendation

To address this, an on-chain view function should be implemented to allow re-
layers to see the exact fee amount they will receive before executing a withdrawal transaction.

## Team Response

Fixed.

---
