# Cluster -1140

**Rank:** #126  
**Count:** 120  

## Label
Inconsistent or incorrect fee calculation due to improper rounding, missing logic paths, or flawed input handling, leading to revenue loss, free swaps, or denial-of-service via invalid transaction validation.

## Cluster Information
- **Total Findings:** 120

## Examples

### Example 1

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
### Example 2

**Auto Label:** Retroactive or incorrect fee application due to lack of timestamped, versioned, or settled fee states, leading to financial misalignment and revenue loss.  

**Original Text Preview:**

## Severity: Low Risk

## Context
BaseFeeCalculator.sol#L56-L57

## Description
Changing the protocol fee will affect all past unclaimed fees, including those already finalized, not only the fees accrued after the change. Consider the following example:

1. The protocol fee is set to 10%. The accountant submits a snapshot, and the owner agrees with it.
2. The dispute period has passed.
3. The protocol fee is changed to 20%, which immediately applies to the unclaimed fees.
4. The protocol fees are claimed before the owner notices the change.

## Recommendation
To be accurate, fees finalized before the protocol fee change should still use the old value. This could be a protocol design choice since it would require keeping track of the timestamps of each protocol fee change, which could introduce additional complexity to the code.

If no change to the code is made, one mitigation could be encouraging fee recipients to claim fees as soon as possible, or requiring the protocol fee change to use a timelock so that vault owners will be aware in advance.

## Aera
Acknowledged. This is part of the trust model to keep the code simple. We plan to carefully communicate any protocol fee changes so that fees can be claimed in advance.

## Spearbit
Acknowledged.

---
### Example 3

**Auto Label:** Inaccurate interest rate or debt calculation due to flawed state updates, race conditions, or incorrect math, leading to erroneous profit/loss, inflated debt, or unintended interest accrual.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/1019 

## Found by 
0x23r0, 0xAlix2, 0xEkko, 0xc0ffEE, 0xgee, Brene, DharkArtz, Drynooo, Etherking, FalseGenius, HeckerTrieuTien, Hueber, Kirkeelee, Kvar, PNS, Rorschach, Ruppin, SafetyBytes, Sir\_Shades, Smacaud, Sparrow\_Jac, Tigerfrake, Uddercover, Z3R0, anchabadze, coin2own, crazzyivan, future, ggg\_ttt\_hhh, gkrastenov, good0000vegetable, h2134, jokr, khaye26, kom, newspacexyz, oxelmiguel, patitonar, theboiledcorn, theweb3mechanic, udo, wickie, ydlee, zraxx

### Summary

Double interest applied in borrowed amount calculation causes inflated borrow values and incorrect liquidation checks.

### Root Cause

An insufficient shortfall is checked when `borrowedAmount > collateral`. The  `borrowedAmount` is calculated inside the `liquidateBorrowAllowedInternal `function as follows:

https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L348
```solidity
 borrowedAmount =
                (borrowed * uint256(LTokenInterface(lTokenBorrowed).borrowIndex())) / borrowBalance.borrowIndex;
```

The borrowed value passed into this function comes from `liquidateBorrow`, where it is calculated using `getHypotheticalAccountLiquidityCollateral`:

```solidity
        (uint256 borrowed, uint256 collateral) =
            lendStorage.getHypotheticalAccountLiquidityCollateral(borrower, LToken(payable(borrowedlToken)), 0, 0);

        liquidateBorrowInternal(
            msg.sender, borrower, repayAmount, lTokenCollateral, payable(borrowedlToken), collateral, borrowed
        );
```

The function` getHypotheticalAccountLiquidityCollateral` already returns the borrowed amount with accrued interest included, since it internally uses the `borrowWithInterestSame` and `borrowWithInterest` methods.

As a result, when the protocol calculates shortfall, it applies interest a second time by again multiplying the already interest-included borrowed amount by the current borrow index and dividing by the stored index.

This causes the `borrowedAmount` used for shortfall checks to be inflated beyond the actual debt.
### Internal Pre-conditions

N/A

### External Pre-conditions

N/A

### Attack Path

N/A

### Impact

Users may be incorrectly flagged for liquidation due to an artificially high perceived borrow amount.

### PoC

_No response_

### Mitigation

Avoid double-applying interest. If the borrowed amount already includes accrued interest, it should not be adjusted again using the borrow index.

---
