# Cluster -1187

**Rank:** #69  
**Count:** 249  

## Label
Failure to account for accrued interest in rate calculations and state updates leads to mispriced rates, retroactive rate application, and inaccurate utilization, enabling financial exploitation and protocol instability.

## Cluster Information
- **Total Findings:** 249

## Examples

### Example 1

**Auto Label:** Failure to account for accrued interest in rate calculations and state updates leads to mispriced rates, retroactive rate application, and inaccurate utilization, enabling financial exploitation and protocol instability.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

The function `rebase()` handles the share price increase to apply the desired APR. The issue is that the code doesn't use `lastRebaseTime` when calculating rate increase and it assumes 24 hours have passed from the last time which couldn't not be the case. This would wrong APR for some of the time. For example if `rebase()` is called 1 hour sooner or later then for those 1 hour the share price rate would be wrong.

## Recommendations

Use `duration = block.timestamp - lastRebaseTime` to calculate the real rate increase that accrued from the last rebase call.

---
### Example 2

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
### Example 3

**Auto Label:** Failure to account for accrued interest in rate calculations and state updates leads to mispriced rates, retroactive rate application, and inaccurate utilization, enabling financial exploitation and protocol instability.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Low

## Description

The `PositionManager.setInterestRateStrategy()` function enables the protocol owner to update the interest rate strategy contract. However, this operation does not include any mechanism to migrate or initialize the internal state required by the new strategy. Parameters such as `targetUtilization`, `endRateAt`, and `lastUpdate` for each vault remain uninitialized in the new strategy, which leads to inaccurate and inflated interest rate calculations.

```solidity
function setInterestRateStrategy(address _newInterestRateStrategy) external onlyOwner {
    if (_newInterestRateStrategy == address(0)) {
        revert ZeroAddress();
    }

    emit NewInterestRateStrategy(address(interestRateStrategy), address(_newInterestRateStrategy));

    interestRateStrategy = IInterestRateStrategy(_newInterestRateStrategy);
}
```

The interest rate calculation function, `interestRate()`, relies on vault-specific parameters to compute a smooth and adaptive rate curve based on the difference between current utilization and the target utilization. If these parameters are missing or zero-initialized, the function calculates a large error (`err`), resulting in excessive linear adaptation and an artificially high interest rate.

However, this function is strictly controlled by the owner and is expected to be used only in exceptional circumstances.

## Recommendation

The new `interestRateStrategy` contract should either be initialized with the current state of each vault or properly migrate relevant parameters from the previous strategy to ensure accuracy and consistency in interest rate calculations.

---
