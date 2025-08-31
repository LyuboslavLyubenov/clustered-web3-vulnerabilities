# Cluster -1067

**Rank:** #350  
**Count:** 17  

## Label
Flawed comparison logic in post-only order validation enables unintended executions and unauthorized commission payments, leading to financial loss and violated order behavior.

## Cluster Information
- **Total Findings:** 17

## Examples

### Example 1

**Auto Label:** Flawed comparison logic in post-only order validation enables unintended executions and unauthorized commission payments, leading to financial loss and violated order behavior.  

**Original Text Preview:**

The `CowSwapFiller` contract validates orders with a strict requirement that the order's sell amount must be less than or equal to the filler's current sell amount:

```solidity
function isValidSignature(bytes32 orderHash, bytes calldata signature) external view returns (bytes4) {
    --- SNIPPED ---

    uint256 orderPrice = Math.mulDiv(order.buyAmount, D27, order.sellAmount, Math.Rounding.Floor);
@>  require(order.sellAmount <= sellAmount && orderPrice >= price, CowSwapFiller__OrderCheckFailed(100));

    --- SNIPPED ---
}
```

This creates a timing issue between the time an order is created (off-chain via the CoW Protocol API) and the time it is settled on-chain:

- When a bot submits a CoW Protocol order, it typically uses `Folio.getBid()` to fetch the current `sellAmount` and `buyAmount`.
- Between the order creation and settlement, the available `sellAmount` can change (e.g., due to other bids or redemptions).
- If the `sellAmount` decreases, the settlement will fail even if the order price is still favorable and could be partially filled.

**Consider the following scenarios:**

1. `AUCTION_LAUNCHER` launches an auction to rebalance the basket from 1000 ETH to 500 ETH, and buy 1,500,000 USDT (start price: `3000 USDT/ETH`).
2. A bot listens to the auction event and then submits an order through the CoW Protocol API for `500 ETH → 1,500,000 USDT` using current values from `Folio.getBid()`.
3. Before settlement, other bidders consume part of the available ETH. The filler now only has `400 ETH` available.
4. During on-chain settlement:

   - The filler contract is created via `pre-hook` with `sellAmount = 400 ETH` and `buyAmount = 1,199,600 USDT` (at the decayed auction price).
   - So the price in the filler is set to `2999 USDT/ETH`.
   - The settlement attempts to validate the original order with `order.sellAmount = 500`.
   - The signature validation fails because `order.sellAmount: 500 <= sellAmount: 400` is `false`, even though the order price (`3000`) is better than the current price (`2999`).

5. Even if the solver could partially fill the order (e.g., with `200 ETH` for `~600,000 USDT` at the requested price) at that block, this would still be a net positive execution at a better price, but it is blocked by the above validation.

Below is an example where an order could not be fully satisfied in a single settlement and was instead partially filled:
https://explorer.cow.fi/orders/0x358c3cf03715224c26f8043c94502c4649e8c59e94d43f16a90e2f8c72e3ad41a53a13a80d72a855481de5211e7654fabdfe3526685b2543/?tab=fills

**Recommendation:**

Given the limited `sellToken` pull via approval and that `orderPrice` already enforces the acceptable price, factoring in both `order.buyAmount` and `order.sellAmount`, it would be appropriate to rely solely on `orderPrice` for verification and remove the `order.sellAmount <= sellAmount` check.

---
### Example 2

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
### Example 3

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
