# Cluster -1161

**Rank:** #265  
**Count:** 32  

## Label
Missing initialization or setter functions leads to permanent, unchangeable default values, causing critical flaws in fee management, protocol functionality, and revenue control.

## Cluster Information
- **Total Findings:** 32

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

**Auto Label:** Missing initialization or setter functions leads to permanent, unchangeable default values, causing critical flaws in fee management, protocol functionality, and revenue control.  

**Original Text Preview:**

The `Minter` contract has a `MAX_REBASE_RATE` constant. Additionally, on initialization of the `rebaseRate` variable, there is a comment that says "30% bootstrap". This suggests that the rebase rate is intended to be adjustable, but there is no setter function provided to change it after deployment.

If the rebase rate is meant to be adjustable, it should have a setter function that allows the owner to change it.

---
### Example 3

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
