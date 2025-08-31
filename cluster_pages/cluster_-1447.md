# Cluster -1447

**Rank:** #153  
**Count:** 91  

## Label
Inconsistent or incorrect precision handling in financial calculations leads to flawed value computations, risking under-collateralization, erroneous liquidations, and compromised asset valuation across systems.

## Cluster Information
- **Total Findings:** 91

## Examples

### Example 1

**Auto Label:** **Decimal precision errors in arithmetic and comparisons lead to incorrect valuations, inflated values, or silent overflows, compromising financial accuracy and system integrity.**  

**Original Text Preview:**

In the `DecreasePositionSizeUtils` and `BorrowingFeesUtils` contracts, there are potential rounding errors that can result in users being favored by a better liquidation price. Specifically, in the `prepareCallbackValues` function of `DecreasePositionSizeUtils`, the calculation of `newLiqPrice` involves a division operation that can lead to rounding errors, then in the `getTradeLiquidationPrice` function of `BorrowingFeesUtils`, the calculation can also result in rounding errors.
DecreasePositionSizeUtils.sol

```solidity
    function prepareCallbackValues(
        ITradingStorage.Trade memory _existingTrade,
        ITradingStorage.Trade memory _partialTrade,
        ITradingCallbacks.AggregatorAnswer memory _answer
    ) internal view returns (IUpdatePositionSizeUtils.DecreasePositionSizeValues memory values) {
<...>
        // 6. Calculate existing and new trade liquidation price
        values.existingLiqPrice = TradingCommonUtils.getTradeLiquidationPrice(_existingTrade, _answer.current);
        values.newLiqPrice = TradingCommonUtils.getTradeLiquidationPrice(
            _existingTrade,
            _existingTrade.openPrice,
            values.newCollateralAmount,
            values.newLeverage,
            values.isLeverageUpdate ? int256(values.closingFeeCollateral) - values.pnlToRealizeCollateral : int256(0),
            _getMultiCollatDiamond().getTradeLiquidationParams(_existingTrade.user, _existingTrade.index),
            _answer.current,
            values.isLeverageUpdate
                ? 1e18
>>              : ((values.existingPositionSizeCollateral - values.positionSizeCollateralDelta) * 1e18) /
                    values.existingPositionSizeCollateral,
            false
        );


        /// @dev Other calculations are in separate helper called after realizing pending holding fees
        // Because they can be impacted by pending holding fees realization (eg. available in diamond calc)
    }
```

BorrowingFeesUtils.sol

```solidity
    function getTradeLiquidationPrice(IBorrowingFees.LiqPriceInput calldata _input) internal view returns (uint256) {
        uint256 closingFeesCollateral = TradingCommonUtils.getTotalTradeFeesCollateral(
            _input.collateralIndex,
            address(0), // never apply fee tiers
            _input.pairIndex,
            (_input.collateral * _input.leverage) / 1e3,
            _input.isCounterTrade
        );


        IFundingFees.TradeHoldingFees memory holdingFees;
        int256 totalRealizedPnlCollateral;


        if (!_input.beforeOpened) {
            holdingFees = _getMultiCollatDiamond().getTradePendingHoldingFeesCollateral(
                _input.trader,
                _input.index,
                _input.currentPairPrice
            );


            (, , totalRealizedPnlCollateral) = _getMultiCollatDiamond().getTradeRealizedPnlCollateral(
                _input.trader,
                _input.index
            );
        }


        uint256 spreadP = _getMultiCollatDiamond().pairSpreadP(_input.trader, _input.pairIndex);
        ITradingStorage.ContractsVersion contractsVersion = _input.beforeOpened
            ? _getMultiCollatDiamond().getCurrentContractsVersion()
            : _getMultiCollatDiamond().getTradeContractsVersion(_input.trader, _input.index);


        return
            _getTradeLiquidationPrice(
                _input.openPrice,
                _input.long,
                _input.collateral,
                _input.leverage,
                int256(closingFeesCollateral) +
                    ((holdingFees.totalFeeCollateral - totalRealizedPnlCollateral) *
>>                      int256(_input.partialCloseMultiplier)) /
                    1e18 +
                    _input.additionalFeeCollateral,
                _getMultiCollatDiamond().getCollateral(_input.collateralIndex).precisionDelta,
                _input.liquidationParams,
                contractsVersion,
                spreadP
            );
    }
```

Consider rounding the `_feesCollateral` value towards positive infinity.

---
### Example 2

**Auto Label:** **Decimal precision errors in arithmetic and comparisons lead to incorrect valuations, inflated values, or silent overflows, compromising financial accuracy and system integrity.**  

**Original Text Preview:**

The implied volatility plays a crucial role in the Black-Scholes equation to calculate the option prices:

For a **European call option**:

C = S₀ × N(d₁) − K × e^(−rT) × N(d₂)

For a **European put option**:

P = K × e^(−rT) × N(−d₂) − S₀ × N(−d₁)

Where:

- (C) = Call option price
- (P) = Put option price
- (S₀) = Current price of the underlying asset
- (K) = Strike price
- (r) = Risk-free interest rate (annualized)
- (T) = Time to maturity (in years)
- (σ) = Volatility of the underlying asset (annualized)
- (N(x)) = Cumulative distribution function (CDF) of the standard normal distribution

Also, the coefficients (d₁) and (d₂):

d₁ = [ln(S₀ / K) + (r + (½) × σ²) × T] / (σ × √T),  
d₂ = d₁ − σ × √T

As it is clear from the abovementioned formula, if we decrease the σ (volatility), the d₁ and d₂ increase respectively. However, the key point here is that even if N(x) increases, the strike N(d₂) increases more than the spot N(d₁), and the overall premium decreases.

Also, in the case where N(d₂) becomes more than N(d₁), the contract makes the premium become zero:

````Solidity
        call = strikeNd2 <= spotNd1 ? spotNd1 - strikeNd2 : 0

---
### Example 3

**Auto Label:** **Decimal precision errors in arithmetic and comparisons lead to incorrect valuations, inflated values, or silent overflows, compromising financial accuracy and system integrity.**  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

The protocol applies the same strike price rounding logic to both `CALL` and `PUT` options. While this works in favor of the protocol for `CALL` options, it reduces safety for `PUT` options.

Specifically, the strike price is first computed by applying a relative discount to the current price (for PUTs), and then passed to `_roundPrice()`, which rounds **upward** if the remainder exceeds the midpoint. This pushes the strike **closer to or into the money**, reducing the out-of-the-money (OTM) buffer intended for PUT options.

```solidity
function _calculateStrike(uint256 preciseCurrentPrice, uint256 relativeStrike) internal view returns (uint256) {
    uint256 calculatedStrike = preciseCurrentPrice - ((preciseCurrentPrice * relativeStrike) / 100);
@>  return _roundPrice(calculatedStrike);
}
```

```solidity
function _roundPrice(uint256 price) internal view returns (uint256) {
    RoundingOption currentRoundingOption_ = StrategyStorage.layout().setUp.base.currentRoundingOption;
    uint256 remainder;
    uint256 roundedDown;
    if (currentRoundingOption_ == RoundingOption.NEAREST_HUNDRED) {
        remainder = price % (100 * 1e18);
        roundedDown = price - remainder;
@>      if (remainder >= 50 * 1e18) return roundedDown + (100 * 1e18);
        else return roundedDown;
    } else if (currentRoundingOption_ == RoundingOption.NEAREST_TEN) {
        remainder = price % (10 * 1e18);
        roundedDown = price - remainder;
@>      if (remainder >= 5 * 1e18) return roundedDown + (10 * 1e18);
        else return roundedDown;
    } else {
        return price;
    }
}
```

Consider this scenario with `NEAREST_HUNDRED` rounding:
**Scenario 1:**

- Spot price: 1000 USD.
- Relative strike: 5%.
- Raw PUT strike = 950 USD.
- Rounded strike = 1000 USD (ATM instead of OTM).

**Scenario 2:**

- Spot price: 3500 USD.
- Relative strike: 10%.
- Raw PUT strike = 3150 USD.
- Rounded strike = 3200 USD (300 (~8.25% relative strike) OTM instead of 350 OTM).

In both cases, upward rounding reduces the distance between the strike and spot price for PUT positions.

## Recommendation

Use different rounding directions for `CALL` and `PUT` options to reflect their opposite risk implications:

- For `CALL` options, keep rounding strike prices up, since that makes them more out-of-the-money and benefits the protocol.
- For `PUT` options, round down instead, so the strike stays further from the current price and doesn’t unintentionally favor the trader.

---
