# Cluster -1036

**Rank:** #74  
**Count:** 228  

## Label
Insufficient or incorrect accounting of user funds leads to double withdrawals, unaccounted surpluses, and unauthorized borrowing, causing severe user losses and enabling attacker exploitation through flawed state tracking and validation.

## Cluster Information
- **Total Findings:** 228

## Examples

### Example 1

**Auto Label:** Improper limit validation due to flawed state tracking, leading to unjustified rejections of stakes and asset locking.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

The `StakingManager` contract implements a staking limit mechanism through the `stakingLimit` parameter to control the maximum amount of HYPE tokens that can be staked in the system. However, the current implementation fails to accurately track the actual amount of tokens under management due to not accounting for withdrawn tokens.

The `stake()` function performs a limit check using:

```solidity
require(totalStaked + msg.value <= stakingLimit, "Staking limit reached");
```

However, `totalStaked` is only incremented when users stake and never decremented, while a separate `totalClaimed` tracks withdrawals. This means the actual amount of tokens under management is `totalStaked - totalClaimed`, but the limit check uses the raw `totalStaked` value.

This creates a situation where the contract could reject new stakes even when the actual amount under management is well below the limit, effectively reducing the protocol's capacity unnecessarily.

### Proof of Concept

- Admin sets stakingLimit to 1000 HYPE
- Alice stakes 1000 HYPE (totalStaked = 1000)
- Alice withdraws 500 HYPE (totalClaimed = 500, totalStaked = 1000)
- Bob tries to stake 100 HYPE
- Transaction reverts due to "Staking limit reached" even though only 500 HYPE is actually staked

## Recommendations

Update the staking limit check in `stake()` to account for claimed tokens:

```solidity
require(((totalStaked - totalClaimed) + msg.value) <= stakingLimit, "Staking limit reached");
```

---
### Example 2

**Auto Label:** Improper limit validation due to flawed state tracking, leading to unjustified rejections of stakes and asset locking.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

The `StakingManager` contract implements staking limits through three key parameters:
- `stakingLimit`: Maximum total stake allowed (0 = unlimited)
- `minStakeAmount`: Minimum stake per transaction
- `maxStakeAmount`: Maximum stake per transaction (0 = unlimited)

These parameters can be configured by accounts with the `MANAGER_ROLE` through dedicated setter functions. However, there is a logical error in the `setMaxStakeAmount()` function that prevents setting a valid max stake amount when there is no staking limit configured.

```solidity
        if (newMaxStakeAmount > 0) {
            require(newMaxStakeAmount > minStakeAmount, "Max stake must be greater than min");
            require(newMaxStakeAmount < stakingLimit, "Max stake must be less than limit");
        }
```

The issue occurs in the validation logic of `setMaxStakeAmount()`, where it unconditionally requires that any non-zero `newMaxStakeAmount` must be less than `stakingLimit`. This check fails to account for the case where `stakingLimit` is 0 (unlimited), making it impossible to set a max stake amount when no total limit exists. 

### Proof of Concept

1. Initially `stakingLimit = 0` (unlimited total staking)
2. Manager attempts to set `maxStakeAmount = 100 ether` to limit individual transactions
3. The call to `setMaxStakeAmount(100 ether)` reverts because:
   - `newMaxStakeAmount > 0` triggers the validation checks
   - `require(newMaxStakeAmount < stakingLimit)` fails as `100 ether < 0` is false

## Recommendations

Modify the `setMaxStakeAmount()` function to only perform the staking limit validation when a limit is actually set:

```solidity
function setMaxStakeAmount(uint256 newMaxStakeAmount) external onlyRole(MANAGER_ROLE) {
    if (newMaxStakeAmount > 0) {
        require(newMaxStakeAmount > minStakeAmount, "Max stake must be greater than min");
        if (stakingLimit > 0) {
            require(newMaxStakeAmount < stakingLimit, "Max stake must be less than limit");
        }
    }
    maxStakeAmount = newMaxStakeAmount;
    emit MaxStakeAmountUpdated(newMaxStakeAmount);
}
```

---
### Example 3

**Auto Label:** Common vulnerability type: **Improper state validation and exposure management leading to collateral misrepresentation, fund drainage, and protocol insolvency.**  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Medium

## Description

When users request an increase in position size, it will eventually be executed, triggering `executeIncreasePositionSizeMarket`.

```solidity
    function executeIncreasePositionSizeMarket(
        ITradingStorage.PendingOrder memory _order,
        ITradingCallbacks.AggregatorAnswer memory _answer
    ) external {
        // 1. Prepare vars
        ITradingStorage.Trade memory partialTrade = _order.trade;
        ITradingStorage.Trade memory existingTrade = _getMultiCollatDiamond().getTrade(
            partialTrade.user,
            partialTrade.index
        );
        IUpdatePositionSizeUtils.IncreasePositionSizeValues memory values;
        // 2. Refresh trader fee tier cache
        TradingCommonUtils.updateFeeTierPoints(
            existingTrade.collateralIndex,
            existingTrade.user,
            existingTrade.pairIndex,
            0
        );

        // 3. Base validation (trade open, market open)
        ITradingCallbacks.CancelReason cancelReason = _validateBaseFulfillment(existingTrade);

        // 4. If passes base validation, validate further
        if (cancelReason == ITradingCallbacks.CancelReason.NONE) {
            // 4.1 Prepare useful values (position size delta, pnl, fees, new open price, etc.)
>>>         values = IncreasePositionSizeUtils.prepareCallbackValues(existingTrade, partialTrade, _answer);

            // 4.2 Further validation
>>>         cancelReason = IncreasePositionSizeUtils.validateCallback(
                existingTrade,
                values,
                _answer,
                partialTrade.openPrice,
                _order.maxSlippageP
            );

            // 5. If passes further validation, execute callback
            if (cancelReason == ITradingCallbacks.CancelReason.NONE) {
                // 5.1 Update trade collateral / leverage / open price in storage, and reset trade borrowing fees
                IncreasePositionSizeUtils.updateTradeSuccess(_answer.orderId, existingTrade, values, _answer);
                // 5.2 Distribute opening fees and store fee tier points for position size delta
                TradingCommonUtils.processFees(existingTrade, values.positionSizeCollateralDelta, _order.orderType);
            }
        }

        // 6. If didn't pass validation, charge gov fee (if trade exists) and return partial collateral (if any)
        if (cancelReason != ITradingCallbacks.CancelReason.NONE)
>>>         IncreasePositionSizeUtils.handleCanceled(existingTrade, partialTrade, cancelReason, _answer);

        // 7. Close pending increase position size order
        _getMultiCollatDiamond().closePendingOrder(_answer.orderId);

        emit IUpdatePositionSizeUtils.PositionSizeIncreaseExecuted(
            _answer.orderId,
            cancelReason,
            existingTrade.collateralIndex,
            existingTrade.user,
            existingTrade.pairIndex,
            existingTrade.index,
            existingTrade.long,
            _answer.current,
            _getMultiCollatDiamond().getCollateralPriceUsd(existingTrade.collateralIndex),
            partialTrade.collateralAmount,
            partialTrade.leverage,
            values
        );
    }
```

Inside `executeIncreasePositionSizeMarket`, `prepareCallbackValues` is called to calculate all values required for increasing the position size, including `values.counterTradeCollateralToReturn` if the position has `isCounterTrade` set to true. In this case, `_partialTrade.collateralAmount` will also be decreased by `values.counterTradeCollateralToReturn`.

```solidity
    function prepareCallbackValues(
        ITradingStorage.Trade memory _existingTrade,
        ITradingStorage.Trade memory _partialTrade,
        ITradingCallbacks.AggregatorAnswer memory _answer
    ) internal view returns (IUpdatePositionSizeUtils.IncreasePositionSizeValues memory values) {
        // 1.1 Calculate position size delta
        bool isLeverageUpdate = _partialTrade.collateralAmount == 0;
        values.positionSizeCollateralDelta = TradingCommonUtils.getPositionSizeCollateral(
            isLeverageUpdate ? _existingTrade.collateralAmount : _partialTrade.collateralAmount,
            _partialTrade.leverage
        );

        // 1.2 Validate counter trade and update position size delta if needed
        if (_existingTrade.isCounterTrade) {
>>>         (values.isCounterTradeValidated, values.exceedingPositionSizeCollateral) = TradingCommonUtils
                .validateCounterTrade(_existingTrade, values.positionSizeCollateralDelta);

            if (values.isCounterTradeValidated && values.exceedingPositionSizeCollateral > 0) {
                if (isLeverageUpdate) {
                    // For leverage updates, simply reduce leverage delta to reach 0 skew
                    _partialTrade.leverage -= uint24(
                        (values.exceedingPositionSizeCollateral * 1e3) / _existingTrade.collateralAmount
                    );
                } else {
                    // For collateral adds, reduce collateral delta to reach 0 skew
                    values.counterTradeCollateralToReturn =
                        (values.exceedingPositionSizeCollateral * 1e3) /
                        _partialTrade.leverage;
>>>                 _partialTrade.collateralAmount -= uint120(values.counterTradeCollateralToReturn);
                }

                values.positionSizeCollateralDelta = TradingCommonUtils.getPositionSizeCollateral(
                    isLeverageUpdate ? _existingTrade.collateralAmount : _partialTrade.collateralAmount,
                    _partialTrade.leverage
                );
            }
        }

        // ...
    }
```

However, if `validateCallback` return non `CancelReason.NONE` and `handleCanceled` is triggered. it will only send `_partialTrade.collateralAmount` to the user and not considering `values.counterTradeCollateralToReturn`.

```solidity
    function handleCanceled(
        ITradingStorage.Trade memory _existingTrade,
        ITradingStorage.Trade memory _partialTrade,
        ITradingCallbacks.CancelReason _cancelReason,
        ITradingCallbacks.AggregatorAnswer memory _answer
    ) internal {
        // 1. Charge gov fee on trade (if trade exists)
        if (_cancelReason != ITradingCallbacks.CancelReason.NO_TRADE) {
            // 1.1 Distribute gov fee
            uint256 govFeeCollateral = TradingCommonUtils.getMinGovFeeCollateral(
                _existingTrade.collateralIndex,
                _existingTrade.user,
                _existingTrade.pairIndex
            );
            uint256 finalGovFeeCollateral = _getMultiCollatDiamond().realizeTradingFeesOnOpenTrade(
                _existingTrade.user,
                _existingTrade.index,
                govFeeCollateral,
                _answer.current
            );
            TradingCommonUtils.distributeExactGovFeeCollateral(
                _existingTrade.collateralIndex,
                _existingTrade.user,
                finalGovFeeCollateral
            );
        }

        // 2. Send back partial collateral to trader
        TradingCommonUtils.transferCollateralTo(
            _existingTrade.collateralIndex,
            _existingTrade.user,
>>>         _partialTrade.collateralAmount
        );
    }
```

This will cause the collateral amount returned to users to be inaccurate, resulting in a loss of funds.

## Recommendations

Also consider `values.counterTradeCollateralToReturn` when returning funds inside `handleCanceled`

```diff
//...
        // 2. Send back partial collateral to trader
        TradingCommonUtils.transferCollateralTo(
            _existingTrade.collateralIndex,
            _existingTrade.user,
-            _partialTrade.collateralAmount
+            _partialTrade.collateralAmount + values.counterTradeCollateralToReturn
        );
//...
```

---
