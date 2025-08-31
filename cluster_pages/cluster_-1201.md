# Cluster -1201

**Rank:** #278  
**Count:** 29  

## Label
Precision loss and incorrect validation thresholds due to flawed arithmetic ordering, improper scaling, and inconsistent units, leading to undercharged fees, reverts, or manipulated fee calculations.

## Cluster Information
- **Total Findings:** 29

## Examples

### Example 1

**Auto Label:** Precision loss and incorrect validation thresholds due to flawed arithmetic ordering, improper scaling, and inconsistent units, leading to undercharged fees, reverts, or manipulated fee calculations.  

**Original Text Preview:**

In the `_calculateFeesUSD` function of `OperationalTreasury.sol`, the fee calculation is performed in a way that may lead to precision loss due to **Division Before Multiply**. The current implementation:

```solidity
fees = price.mulDivRoundingUp(baseSetUp.protocolFees, FEES_UNIT);
fees = (fees * amount).toDecimals(36, 18);
```

The issue arises because the protocol fees are divided by `FEES_UNIT` before being multiplied by the `amount`. This order of operations can result in rounding down of intermediate values, **potentially leading to undercharging of fees.**

To minimize precision loss, the calculation should be reordered. The recommended implementation should be:

```solidity
fees = (price * amount).mulDivRoundingUp(baseSetUp.protocolFees, FEES_UNIT);
fees = fees.toDecimals(36, 18);
```

---
### Example 2

**Auto Label:** Underflow errors from unchecked arithmetic in fee, balance, and share calculations, leading to reverts, fund loss, or incorrect state due to invalid or unvalidated inputs.  

**Original Text Preview:**

##### Description
An attacker can directly transfer tokens to the Umbrella contract, causing an underflow error when `Umbrella._setPendingDeficit()` or `Umbrella._setDeficitOffset()` is triggered.

This happens because the `_coverDeficit()` function returns the entire contract balance when transferring tokens, rather than just the portion sent by `msg.sender`, and hacker can increase it by a direct transfer:
```solidity
IERC20(aToken).safeTransferFrom(_msgSender(), address(this), amount);
amount = IERC20(aToken).balanceOf(address(this));
```
https://github.com/bgd-labs/aave-umbrella-private/blob/946a220a57b4ae0ad11d088335f9bcbb0e34dcef/src/contracts/umbrella/Umbrella.sol#L148-L151

Governance has the ability to transfer excess tokens via an emergency function. However, an attacker can frontrun the next call to `coverDeficitOffset()` and `coverPendingDeficit()` functions, causing a DoS again.

##### Recommendation
We recommend considering the case where the contract already holds funds when calculating the actual received balance.

***

---
### Example 3

**Auto Label:** Stale state and improper state synchronization lead to incorrect fee calculations, invalid arithmetic, and unintended liquidations due to flawed control flow and unupdated balance tracking.  

**Original Text Preview:**

**Description:** When a user deposits directly into `Manager::deposit`, the protocol fee is calculated via the [`Manager::_transferFee`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/Manager.sol#L226-L242) function:

```solidity
function _transferFee(address _yToken, uint256 _shares, uint256 _fee) internal returns (uint256) {
    if (_fee == 0) {
        return _shares;
    }
    uint256 feeShares = (_shares * _fee) / Constants.HUNDRED_PERCENT;

    IERC20(_yToken).safeTransfer(treasury, feeShares);

    return feeShares;
}
```

The issue is that when `_fee == 0`, the function returns the full `_shares` amount instead of returning `0`. This leads to incorrect logic downstream in [`Manager::_deposit`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/Manager.sol#L286-L296), where the result is subtracted from the total shares:

```solidity
// transfer fee to treasury, already applied on adjustedShares
uint256 adjustedFeeShares = _transferFee(order.yToken, adjustedShares, _fee);

// Calculate adjusted gas fee shares
uint256 adjustedGasFeeShares = (_gasFeeShares * order.exchangeRateInUnderlying) / currentExchangeRate;

// transfer gas to caller
IERC20(order.yToken).safeTransfer(_caller, adjustedGasFeeShares);

// remaining shares after gas fee
uint256 sharesAfterAllFee = adjustedShares - adjustedFeeShares - adjustedGasFeeShares;
```

If `_fee == 0`, the `adjustedFeeShares` value will incorrectly equal `adjustedShares`, causing `sharesAfterAllFee` to underflow (revert), assuming `adjustedGasFeeShares` is non-zero.

**Impact:** Deposits into the `Manager` contract with a fee of zero will revert if any gas fee is also deducted. In the best-case scenario, the deposit fails. In the worst case—if the subtraction somehow passes unchecked—it could result in zero shares being credited to the user.

**Recommended Mitigation:** Update `_transferFee` to return `0` when `_fee == 0`, to ensure downstream calculations behave correctly:

```diff
  if (_fee == 0) {
-     return _shares;
+     return 0;
  }
```

**YieldFi:** Fixed in commit [`6e76d5b`](https://github.com/YieldFiLabs/contracts/commit/6e76d5beee3ba7a49af6becc58a596a4b67841c3)

**Cyfrin:** Verified. `_transferFee` now returns `0` when `_fee = 0`

---
