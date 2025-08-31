# Cluster -1397

**Rank:** #213  
**Count:** 51  

## Label
Missing or flawed validation of critical thresholds leads to incorrect state calculations, enabling unauthorized exposure, under-collateralization, and financial loss through improper bounds checking and logic errors.

## Cluster Information
- **Total Findings:** 51

## Examples

### Example 1

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
### Example 2

**Auto Label:** Missing or flawed validation of critical thresholds leads to incorrect state calculations, enabling unauthorized exposure, under-collateralization, and financial loss through improper bounds checking and logic errors.  

**Original Text Preview:**

Due to the lack of validation for `tradeOwnership`, it is possible for the total `tradeOwnership` in the external trade to be either lower or greater than `_WEIGHT_PRECISION`, which could break the asset balance calculation within the baskets. Consider verifying that the total `tradeOwnershi`p in each external trade is equal to `_WEIGHT_PRECISION`.

---
### Example 3

**Auto Label:** Missing or flawed validation of critical thresholds leads to incorrect state calculations, enabling unauthorized exposure, under-collateralization, and financial loss through improper bounds checking and logic errors.  

**Original Text Preview:**

[link](https://github.com/Storm-Labs-Inc/cove-contracts-core/blob/6b607d137f898c0f421b4ba4e748f41b09b41518/src/libraries/BasketManagerUtils.sol#L827)
`trade.maxAmount` appears to be designed to protect the counterparty (`toBasket`).

- `fromBasket` ultimately receives no less than `trade.minAmount` of `buyToken`.
- `toBasket` pays no more than `trade.maxAmount` of `buyToken`.

Therefore, it should be compared with `initialBuyAmount` rather than `info.netBuyAmount`.

---
