# Cluster -1199

**Rank:** #267  
**Count:** 31  

## Label
Failure to validate real-time price movements or transaction logic leads to exploitable price manipulation, missed liquidations, and risk-free profit extraction through timing and market abuse.

## Cluster Information
- **Total Findings:** 31

## Examples

### Example 1

**Auto Label:** Misordered price impact calculations bypass stop losses and enable unfavorable execution prices, leading to user losses and flawed trade enforcement.  

**Original Text Preview:**

When users set a stop loss, they expect their positions to be closed at or near the specified price level. However, the current implementation can potentially result in an execution price that is significantly worse than the user-specified stop loss level. It's because the `priceAfterImpact` calculation, which includes both spread and price impact, is applied after the stop loss is triggered.

It's recommended to allow users to set stop loss as the price after close spread and price impact calculation.

```solidity
    function executeTriggerCloseOrderCallback(
        ITradingCallbacks.AggregatorAnswer memory _a
    ) internal tradingActivatedOrCloseOnly {
        ...
            if (o.orderType != ITradingStorage.PendingOrderType.LIQ_CLOSE) {
                (, uint256 priceAfterImpact, ) = TradingCommonUtils.getTradeClosingPriceImpact( // @audit execution price can be unfavourable than expected when setting SL
                    ITradingCommonUtils.TradePriceImpactInput(
                        t,
                        v.executionPrice,
                        _a.spreadP,
                        TradingCommonUtils.getPositionSizeCollateral(t.collateralAmount, t.leverage)
                    )
                );
                v.executionPrice = priceAfterImpact;
            }
        ...
    }
```

---
### Example 2

**Auto Label:** Misordered price impact calculations bypass stop losses and enable unfavorable execution prices, leading to user losses and flawed trade enforcement.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

When closing trades or reducing position sizes, the execution price can deviate significantly from expected due to two main factors:

- Spread percentage update by admin.
- Price impact percentage fluctuation.

The `spreadP` and calculated `priceImpactP` used may have changed between when a trade closure is requested and when it's actually executed in the callback. However, there is currently no slippage protection mechanism to prevent trades from being closed at unexpectedly unfavorable prices due to these fluctuations.

This issue also affects the `decreasePositionSize` function in a similar way as `closeTradeMarketCallback`.

```solidity
    function closeTradeMarketCallback(
        ITradingCallbacks.AggregatorAnswer memory _a
    ) internal tradingActivatedOrCloseOnly {
        ...
        (uint256 priceImpactP, uint256 priceAfterImpact, ) = TradingCommonUtils.getTradeClosingPriceImpact( // @audit price impact can fluctuate due to open interest and depth
            ITradingCommonUtils.TradePriceImpactInput(
                t,
                _a.price,
                _a.spreadP, // @audit spreadP can be updated between closeTrade and closeTradeMarketCallback
                TradingCommonUtils.getPositionSizeCollateral(t.collateralAmount, t.leverage)
            )
        );
        ...
    }
```

## Recommendations

- Allow users to specify a maximum acceptable slippage when requesting to close a trade.

---
### Example 3

**Auto Label:** Misordered price impact calculations bypass stop losses and enable unfavorable execution prices, leading to user losses and flawed trade enforcement.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** High

## Description

The protocol implements different spread and price impact charging mechanisms for trades opened before and after v9.2. However, inconsistency in these mechanisms allows users to exploit pre-v9.2 trades to avoid full spread and price impact charges when modifying their positions.

When increasing position sizes, the spread and price impact percentage are charged 50% regardless of trade opened before or after v9.2.

```solidity
    function prepareCallbackValues(
        ITradingStorage.Trade memory _existingTrade,
        ITradingStorage.Trade memory _partialTrade,
        ITradingCallbacks.AggregatorAnswer memory _answer
    ) internal view returns (IUpdatePositionSizeUtils.IncreasePositionSizeValues memory values) {
        ...
        // 3. Calculate price impact values
        (, values.priceAfterImpact) = _getMultiCollatDiamond().getTradePriceImpact(
            TradingCommonUtils.getMarketExecutionPrice(_answer.price, _answer.spreadP, _existingTrade.long, true), // @audit charge half of spread and price impact regardless of trade opened before or after v9.2
            _existingTrade.pairIndex,
            _existingTrade.long,
            _getMultiCollatDiamond().getUsdNormalizedValue(
                _existingTrade.collateralIndex,
                values.positionSizeCollateralDelta
            ),
            false,
            true,
            0
        );
        ...
    }
```

However, when reducing position size or close trade, if the trade opened before v9.2, no spread or price impact is charged.

```solidity
function getTradeClosingPriceImpact(
        ITradingCommonUtils.TradePriceImpactInput memory _input
    ) external view returns (uint256 priceImpactP, uint256 priceAfterImpact, uint256 tradeValueCollateralNoFactor) {
        ITradingStorage.Trade memory trade = _input.trade;

        // 0. If trade opened before v9.2, return market price (no closing spread or price impact)
        if (_getMultiCollatDiamond().getTradeLiquidationParams(trade.user, trade.index).maxLiqSpreadP == 0) {
            return (0, _input.marketPrice, 0);
        }
        ...
    }
```

This inconsistency creates an exploit:

- Users with open trades from before v9.2 can increase their position size, paying only 50% of spread and price impact.
- They can then decrease their position size or close the trade without any spread or price impact charges.

## Recommendations

When increasing position sizes, charge the 100% spread and price impact percentage if the trade was opened before v9.2.

---
