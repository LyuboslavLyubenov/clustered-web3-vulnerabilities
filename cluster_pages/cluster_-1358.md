# Cluster -1358

**Rank:** #68  
**Count:** 259  

## Label
Inadequate input validation and arithmetic safeguards lead to incorrect state transitions, financial misalignment, and potential overflow or underflow, enabling malicious actors to manipulate fees, liquidity, or ratios.

## Cluster Information
- **Total Findings:** 259

## Examples

### Example 1

**Auto Label:** Missing input or state validation leads to exploitable fee or parameter manipulation, enabling incorrect logic, overflow, or unfair economic outcomes.  

**Original Text Preview:**

In the `updateFeeToken`, `updateFeeAmount`, and `updateFeeTo` functions, the fee config is not checked to ensure it has been properly set beforehand. As a result, an editor could mistakenly update the fee config for a chain ID where the fee config has not been set.

Example of impact: If the editor updates the fee token, the fee config could not be set for that chain ID.

---
### Example 2

**Auto Label:** Missing or flawed validation of critical thresholds leads to incorrect state calculations, enabling unauthorized exposure, under-collateralization, and financial loss through improper bounds checking and logic errors.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** High

## Description

Inside the Decrease Position's `updateTradeSuccess`, the closing fee is deducted from the user. However, it is not considered when providing `_values.availableCollateralInDiamond` to the `handleTradeValueTransfer` operation.

```solidity
    function updateTradeSuccess(
        ITradingStorage.Trade memory _existingTrade,
        IUpdatePositionSizeUtils.DecreasePositionSizeValues memory _values,
        ITradingCallbacks.AggregatorAnswer memory _answer
    ) internal {
        // 1. Update trade in storage
        _getMultiCollatDiamond().updateTradePosition(
            ITradingStorage.Id(_existingTrade.user, _existingTrade.index),
            _values.newCollateralAmount,
            _values.newLeverage,
            _existingTrade.openPrice, // open price stays the same
            ITradingStorage.PendingOrderType.MARKET_PARTIAL_CLOSE, // don't refresh liquidation params
            _values.existingPnlCollateral > 0,
            _answer.current
        );
        // 2. Realize trading fees and negative pnl (leverage decrease)
>>>     _getMultiCollatDiamond().realizeTradingFeesOnOpenTrade(
            _existingTrade.user,
            _existingTrade.index,
            _values.closingFeeCollateral +
                (_values.collateralSentToTrader < 0 ? uint256(-_values.collateralSentToTrader) : 0), // reduces collateral available in diamond
            _answer.current
        );

        // 3. Handle collateral/pnl transfers
        TradingCommonUtils.handleTradeValueTransfer(
            _existingTrade,
            _values.collateralSentToTrader,
>>>         int256(_values.availableCollateralInDiamond)
        );
    }
```

This will cause `handleTradeValueTransfer` to process an incorrect amount of collateral that needs to be pulled from the vault.

## Recommendations

Consider including `closingFeeCollateral` when providing `availableCollateralInDiamond` to `handleTradeValueTransfer`.

```diff
// ...
        // 3. Handle collateral/pnl transfers
        TradingCommonUtils.handleTradeValueTransfer(
            _existingTrade,
            _values.collateralSentToTrader,
-            int256(_values.availableCollateralInDiamond)
+            int256(_values.availableCollateralInDiamond - _values.closingFeeCollateral)
        );
// ...
```

---
### Example 3

**Auto Label:** Missing or flawed validation of critical thresholds leads to incorrect state calculations, enabling unauthorized exposure, under-collateralization, and financial loss through improper bounds checking and logic errors.  

**Original Text Preview:**

Due to the lack of validation for `tradeOwnership`, it is possible for the total `tradeOwnership` in the external trade to be either lower or greater than `_WEIGHT_PRECISION`, which could break the asset balance calculation within the baskets. Consider verifying that the total `tradeOwnershi`p in each external trade is equal to `_WEIGHT_PRECISION`.

---
