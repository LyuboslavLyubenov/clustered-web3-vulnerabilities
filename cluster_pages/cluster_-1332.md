# Cluster -1332

**Rank:** #392  
**Count:** 13  

## Label
Inaccurate mathematical calculations in pricing and rate computations due to flawed arithmetic logic or improper precision handling, leading to erroneous asset valuations, flawed trade execution, and potential financial loss.

## Cluster Information
- **Total Findings:** 13

## Examples

### Example 1

**Auto Label:** Incorrect price impact calculation due to flawed state handling or static assumptions, leading to erroneous trade valuation, misaligned position sizing, and potential financial exposure.  

**Original Text Preview:**

`getTradeClosingPriceImpact` is a function that calculates the price impact for a specific trade based on multiple factors, one of which is the total fee in the collateral token (`getTotalTradeFeesCollateral`).

```solidity
    function getTradeClosingPriceImpact(
        ITradingCommonUtils.TradePriceImpactInput memory _input
    ) public view returns (ITradingCommonUtils.TradePriceImpact memory output, uint256 tradeValueCollateralNoFactor) {

        // ...

        tradeValueCollateralNoFactor = getTradeValueCollateral(
            trade,
            getPnlPercent(
                trade.openPrice,
                getPriceAfterImpact(
                    _input.oraclePrice,
                    output.fixedSpreadP + cumulVolPriceImpactP + output.skewPriceImpactP
                ),
                trade.long,
                trade.leverage
            ),
>>>         getTotalTradeFeesCollateral(
                trade.collateralIndex,
                trade.user,
                trade.pairIndex,
                getPositionSizeCollateral(trade.collateralAmount, trade.leverage),
                trade.isCounterTrade
            ),
            _input.currentPairPrice
        );
// ...
}
```

`getTotalTradeFeesCollateral` calculates the fee using the trader's fee multiplier cached in `calculateFeeAmount`. However, in several instances, `getTradeClosingPriceImpact` is called without updating the trade's fee points.

```solidity
    function getTotalTradeFeesCollateral(
        uint8 _collateralIndex,
        address _trader,
        uint16 _pairIndex,
        uint256 _positionSizeCollateral,
        bool _isCounterTrade
    ) public view returns (uint256) {
        uint256 feeRateP = _getMultiCollatDiamond().pairTotalPositionSizeFeeP(_pairIndex);

        uint256 feeRateMultiplier = _isCounterTrade
            ? _getMultiCollatDiamond().getPairCounterTradeFeeRateMultiplier(_pairIndex)
            : 1e3;

        uint256 rawFeeCollateral = (getPositionSizeCollateralBasis(
            _collateralIndex,
            _pairIndex,
            _positionSizeCollateral,
            feeRateMultiplier
        ) * feeRateP) /
            ConstantsUtils.P_10 /
            100 /
            1e3;

        // Fee tier is applied on min fee, but counter trade fee rate multiplier doesn't impact min fee
>>>     return _getMultiCollatDiamond().calculateFeeAmount(_trader, rawFeeCollateral);
    }
```

Here is the list of functions:

`executeManualNegativePnlRealizationCallback` :

```solidity
    function executeManualNegativePnlRealizationCallback(
        ITradingStorage.PendingOrder memory _order,
        ITradingCallbacks.AggregatorAnswer memory _a
    ) external {
        ITradingStorage.Trade memory trade = _getTrade(_order.trade.user, _order.trade.index);

        if (!trade.isOpen) return;

        (ITradingCommonUtils.TradePriceImpact memory priceImpact, ) = TradingCommonUtils.getTradeClosingPriceImpact(
            ITradingCommonUtils.TradePriceImpactInput(
                trade,
                _a.current,
                TradingCommonUtils.getPositionSizeCollateral(trade.collateralAmount, trade.leverage),
                _a.current,
                false // don't use cumulative volume price impact
            )
        );
// ..
}
```

`closeTradeMarketCallback` :

```solidity
    function closeTradeMarketCallback(
        ITradingCallbacks.AggregatorAnswer memory _a
    ) external tradingActivatedOrCloseOnly {
        ITradingStorage.PendingOrder memory o = _getPendingOrder(_a.orderId);

        _validatePendingOrderOpen(o);

        ITradingStorage.Trade memory t = _getTrade(o.trade.user, o.trade.index);
        ITradingStorage.TradeInfo memory i = _getTradeInfo(o.trade.user, o.trade.index);

        (ITradingCommonUtils.TradePriceImpact memory priceImpact, ) = TradingCommonUtils.getTradeClosingPriceImpact(
            ITradingCommonUtils.TradePriceImpactInput(
                t,
                _a.current,
                TradingCommonUtils.getPositionSizeCollateral(t.collateralAmount, t.leverage),
                _a.current,
                true
            )
        );
// ...
}
```

`executeTriggerCloseOrderCallback => `validateTriggerCloseOrderCallback` :

```solidity
    function validateTriggerCloseOrderCallback(
        ITradingStorage.Id memory _tradeId,
        ITradingStorage.PendingOrderType _orderType,
        uint64 _open,
        uint64 _high,
        uint64 _low,
        uint64 _currentPairPrice
    ) public view returns (ITradingStorage.Trade memory t, ITradingCallbacks.Values memory v) {
       // ...
        // Return early if trade is not open
        if (v.cancelReason != ITradingCallbacks.CancelReason.NONE) return (t, v);
        v.liqPrice = TradingCommonUtils.getTradeLiquidationPrice(t, _currentPairPrice);
        uint256 triggerPrice = _orderType == ITradingStorage.PendingOrderType.TP_CLOSE
            ? t.tp
            : (_orderType == ITradingStorage.PendingOrderType.SL_CLOSE ? t.sl : v.liqPrice);

        v.exactExecution = triggerPrice > 0 && _low <= triggerPrice && _high >= triggerPrice;
        v.executionPriceRaw = v.exactExecution ? triggerPrice : _open;

        // Apply closing spread and price impact for TPs and SLs, not liquidations (because trade value is 0 already)
        if (_orderType != ITradingStorage.PendingOrderType.LIQ_CLOSE) {
>>>         (v.priceImpact, ) = TradingCommonUtils.getTradeClosingPriceImpact(
                ITradingCommonUtils.TradePriceImpactInput(
                    t,
                    v.executionPriceRaw,
                    TradingCommonUtils.getPositionSizeCollateral(t.collateralAmount, t.leverage),
                    _currentPairPrice,
                    true
                )
            );
            v.executionPrice = v.priceImpact.priceAfterImpact;
        } else {
            v.executionPrice = v.executionPriceRaw;
        }

        // ...
    }
```

**Recommendations**

Consider adding or moving the `updateFeeTierPoints` calls to the start of the operation.

---
### Example 2

**Auto Label:** Incorrect price impact calculation due to flawed state handling or static assumptions, leading to erroneous trade valuation, misaligned position sizing, and potential financial exposure.  

**Original Text Preview:**

The `PriceImpactUtils.setUserPriceImpact` internal function allows a privileged admin to set custom `cumulVolPriceImpactMultiplier` and `fixedSpreadP` for specific trader-pair combinations. The issue with the current implementation is the ability to alter these parameters for a user's `already open positions`, changing the rules of the game post-entry. It can lead to worsens the effective closing price and liquidation price calculations.

Recommendation:
It's recommended to modify the logic so that parameters set via `setUserPriceImpact` only apply to trades opened after the parameters are set for that user-pair.

---
### Example 3

**Auto Label:** Incorrect price impact calculation due to flawed state handling or static assumptions, leading to erroneous trade valuation, misaligned position sizing, and potential financial exposure.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** High

## Description

The `PriceImpactUtils.getTradeSkewPriceImpactP` does not take into account a trade contract version. This causes applying the same skew price impact to all contract versions.
So independent from whether the skew price impact was applied or not when the trade was opened, a half of the price impact will be applied during closing.

```solidity
    function getTradeSkewPriceImpactP(
        uint8 _collateralIndex,
        uint16 _pairIndex,
        bool _long,
        uint256 _tradeOpenInterestUsd, // 1e18 USD
        bool _open
    )
        internal
        view
        returns (
            int256 priceImpactP // 1e10 (%)
        )
    {
        IPriceImpact.PriceImpactValues memory v;

        v.tradePositiveSkew = (_long && _open) || (!_long && !_open);
        v.depth = _getStorage().pairSkewDepths[_collateralIndex][_pairIndex];
        v.tradeSkewMultiplier = v.tradePositiveSkew ? int256(1) : int256(-1);
>>      v.priceImpactDivider = int256(2); // so depth is on same scale as cumul vol price impact

        return
            _getTradePriceImpactP(
                TradingCommonUtils.getPairOiSkewCollateral(_collateralIndex, _pairIndex) *
                    int256(_getMultiCollatDiamond().getCollateralPriceUsd(_collateralIndex)),
                int256(_tradeOpenInterestUsd) * v.tradeSkewMultiplier,
                v.depth,
                ConstantsUtils.P_10,
                ConstantsUtils.P_10
>>          ) / v.priceImpactDivider;
    }
```

## Recommendations

Consider not applying the skew price impact at all when closing trades being opened before the update, i.e. contract version < 10 and applying the full size of the skew price impact when increasing positions for such trades.

---
