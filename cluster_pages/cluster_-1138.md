# Cluster -1138

**Rank:** #73  
**Count:** 233  

## Label
Inconsistent fee handling and data validation across critical functions lead to inaccurate cost estimates, financial loss, and protocol dysfunction due to flawed logic, missing checks, or misaligned state assumptions.

## Cluster Information
- **Total Findings:** 233

## Examples

### Example 1

**Auto Label:** **Missing fee validation or insufficient gas estimation leads to transaction failures, user losses, or incorrect cost projections due to on-chain or off-chain fee mismatches.**  

**Original Text Preview:**

The `OFTTransportAdapter` contract implements the [`_transferViaOFT` function](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/contracts/libraries/OFTTransportAdapter.sol#L57), meant to interact with the OFT messenger and send the funds with that method. To do so, it first [quotes the fees](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/contracts/libraries/OFTTransportAdapter.sol#L85), which are returned as [native or ZRO token fees](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/contracts/interfaces/IOFT.sol#L18-L19). Later, this same output is used as [part of the message](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/contracts/libraries/OFTTransportAdapter.sol#L94).

Even though the `OFTTransportAdapter` contract instructs that it will [pay the fees in native tokens](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/contracts/libraries/OFTTransportAdapter.sol#L85), there is no validation that the value for fees in ZRO tokens is zero, meaning that it will be passed once again to the messenger. This means that if the implementation of the messenger outputs both quotes at the same time, it might not recognize with which asset it will be paid, resulting in a reversion if it tries to get paid with ZRO. As seen in an [example from LayerZero](https://docs.layerzero.network/v2/developers/evm/oft/quickstart#calling-send), in the case of paying in native tokens, the `lzTokenFee` parameter is set to zero.

In order to prevent unexpected outcomes and reversions, especially if the messenger deviates in behavior, consider asserting that the returned `lzTokenFee` is zero when quoting for the cost. Furthermore, consider adding more scenarios to the [mocked contracts](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/contracts/test/MockOFTMessenger.sol#L18-L20) to validate proper integration with protocols.

***Update:** Resolved in [pull request #1029](https://github.com/across-protocol/contracts/pull/1029). The team stated:*

> *Added a zero-check for `lzTokenFee` and added some fee negative-scenario tests (partially addressing M-04).*

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

**Auto Label:** Failure to synchronize critical fee state variables with real-time market conditions, leading to stale calculations, incorrect fee accruals, and transaction failures during active operations.  

**Original Text Preview:**

The `executeManualHoldingFeesRealizationCallback` function calls `resetTradeBorrowingFees` to reset a trade's borrowing fee:

```solidity
    function executeManualHoldingFeesRealizationCallback(
        ITradingStorage.PendingOrder memory _order,
        ITradingCallbacks.AggregatorAnswer memory _a
    ) external {
        ...
        _getMultiCollatDiamond().resetTradeBorrowingFees(
            trade.collateralIndex,
            trade.user,
            trade.pairIndex,
            trade.index,
            trade.long,
            _a.current
        );
        ...
    }
```

The `resetTradeBorrowingFees` function calculates current accumulated fees using `getBorrowingPairPendingAccFees` and stores them as the trade's new initialAccFees:

```solidity
    function resetTradeBorrowingFees(
        uint8 _collateralIndex,
        address _trader,
        uint16 _pairIndex,
        uint32 _index,
        bool _long,
        uint256 _currentPairPrice
    ) public validCollateralIndex(_collateralIndex) {
        uint256 currentBlock = ChainUtils.getBlockNumber();

        (uint64 pairAccFeeLong, uint64 pairAccFeeShort, , ) = getBorrowingPairPendingAccFees(
            _collateralIndex,
            _pairIndex,
            currentBlock,
            _currentPairPrice
        );
        ...

        _getStorage().initialAccFees[_collateralIndex][_trader][_index] = initialFees;

        ...
    }
```

The issue is that this function only updates the trade-specific initialAccFees but does not update the newly calculated accumulated fee back to the global state (`borrowingFeesStorage.pairs`). And the `pairAccFeeLong/pairAccFeeShort` is now affected by the current pair price.

Consider the scenario:

- Manual realization at high price: `executeManualHoldingFeesRealizationCallback` is called when the pair price is high (e.g., 100e10).
- Trade baseline updated: The trade's `initialAccFees` is set based on accumulated fees calculated at the high price.
- Global state remains stale: The pair's global accumulated fees (BorrowingFeesStorage.pairs.accFeeLong/accFeeShort) are NOT updated.
- Price drops significantly: The pair price drops to a much lower value (e.g., 50e10).
- Subsequent fee calculation: When borrowing fees are calculated again, the current global accumulated fees (calculated at the lower price) may be lower than the stored `initialAccFees`.
- Underflow revert: The calculation `current global accumulated fees - initialAccFees` underflows, causing transaction reverts.

It's recommended to update global accumulated fees when `executeManualHoldingFeesRealizationCallback` is called.

---
