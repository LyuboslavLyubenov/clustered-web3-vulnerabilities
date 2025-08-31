# Cluster -1404

**Rank:** #234  
**Count:** 42  

## Label
Inaccurate state calculations due to flawed arithmetic, missing adjustments, or imprecise rounding lead to incorrect debt tracking, compromised financial accuracy, and loss of protocol revenue.

## Cluster Information
- **Total Findings:** 42

## Examples

### Example 1

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
### Example 2

**Auto Label:** Inaccurate interest rate or debt calculation due to flawed state updates, race conditions, or incorrect math, leading to erroneous profit/loss, inflated debt, or unintended interest accrual.  

**Original Text Preview:**

**Details**

[Morpho.sol#L483-L509](https://github.com/morpho-org/morpho-blue/blob/eb65770e12c4cdae5836c0027e35b9c82794c3eb/src/Morpho.sol#L483-L509)

        function _accrueInterest(MarketParams memory marketParams, Id id) internal {
            uint256 elapsed = block.timestamp - market[id].lastUpdate;
            if (elapsed == 0) return;

            if (marketParams.irm != address(0)) {
    @>          uint256 borrowRate = IIrm(marketParams.irm).borrowRate(marketParams, market[id]);
    @>          uint256 interest = market[id].totalBorrowAssets.wMulDown(borrowRate.wTaylorCompounded(elapsed));
                market[id].totalBorrowAssets += interest.toUint128();
                market[id].totalSupplyAssets += interest.toUint128();

                uint256 feeShares;
                if (market[id].fee != 0) {
                    uint256 feeAmount = interest.wMulDown(market[id].fee);
                    // The fee amount is subtracted from the total supply in this calculation to compensate for the fact
                    // that total supply is already increased by the full interest (including the fee amount).
                    feeShares =
                        feeAmount.toSharesDown(market[id].totalSupplyAssets - feeAmount, market[id].totalSupplyShares);
                    position[id][feeRecipient].supplyShares += feeShares;
                    market[id].totalSupplyShares += feeShares.toUint128();
                }

                emit EventsLib.AccrueInterest(id, borrowRate, interest, feeShares);
            }

            // Safe "unchecked" cast.
            market[id].lastUpdate = uint128(block.timestamp);
        }

First let's observe how the original `Morpho` markets integrate with the `AdaptiveIRM`. Notice that the average borrow rate is utilized directly BEFORE any state change has happened.

[AdaptiveIRM.sol#L94-L104](https://github.com/hyperstable/contracts/blob/35db5f2d3c8c1adac30758357fbbcfe55f0144a3/src/libraries/AdaptiveIRM.sol#L94-L104)

        } else {
            int256 linearAdaptation = ADJUSTMENT_SPEED.sMulWad(err) * (int256(block.timestamp) - s.lastUpdate[_vault]);

            if (linearAdaptation == 0) {
                avgRateAtTarget = startRateAtTarget;
                endRateAtTarget = startRateAtTarget;
            } else {
                endRateAtTarget = _newRateAtTarget(startRateAtTarget, linearAdaptation);
                int256 midRateAtTarget = _newRateAtTarget(startRateAtTarget, linearAdaptation / 2);
    @>          avgRateAtTarget = (startRateAtTarget + endRateAtTarget + 2 * midRateAtTarget) / 4;
            }

This is done because the `avgRateAtTarget` is a trapezoidal approximation between the start and end interest rates, which dynamically accounts for the changing interest rate.

[PositionManager.sol#L70-L87](https://github.com/hyperstable/contracts/blob/35db5f2d3c8c1adac30758357fbbcfe55f0144a3/src/core/PositionManager.sol#L70-L87)

        function deposit(address _toVault, uint256 _amountToDeposit) external {
            _isValidVault(_toVault);

            IVault vault = IVault(_toVault);
            IERC20 asset = IERC20(vault.asset());

            asset.transferFrom(msg.sender, address(this), _amountToDeposit);
            uint256 shares = vault.deposit(_amountToDeposit, address(this));

    @>      (, uint256 updatedVaultDebt) = _accruePositionDebt(vault, msg.sender);

            collateralShares[msg.sender][_toVault] += shares;
            uint256 updatedVaultCollateral = vaultCollateral[_toVault] += shares;

    @>      interestRateStrategy.updateVaultInterestRate(
                _toVault, updatedVaultCollateral.mulWad(vault.sharePrice()), updatedVaultDebt, vault.MCR()
            );
        }

Take `PositionManager#deposit` as a comparative example. We see that the position debt is accrued before the interest rate is updated.

[InterestRate.sol#L153-L171](https://github.com/hyperstable/contracts/blob/35db5f2d3c8c1adac30758357fbbcfe55f0144a3/src/libraries/InterestRate.sol#L153-L171)

        function calculateInterestIndex(InterestRateStorage storage s, address _vault)
            internal
            view
            returns (uint256, uint256)
        {
            uint256 interestIndex = s.activeInterestIndex[_vault];
            if (s.lastInterestIndexUpdate[_vault] == block.timestamp) {
                return (interestIndex, 0);
            }

            uint256 factor;
            if (s.interestRate[_vault] > 0) {
                uint256 timeDelta = block.timestamp - s.lastInterestIndexUpdate[_vault];
    @>          factor = timeDelta * s.interestRate[_vault];
                interestIndex += interestIndex.mulDivUp(factor, INTEREST_PRECISION);
            }

            return (interestIndex, factor);
        }

Notice in `InterestRate#calculateInterestIndex` that the cached interest rate is used. This is the stale rate based on the PREVIOUS average interest rate rather than the CURRENT. As a result an incorrect amount of interest will be charge to the users.

**Lines of Code**

[InterestRate.sol#L153-L171](https://github.com/hyperstable/contracts/blob/35db5f2d3c8c1adac30758357fbbcfe55f0144a3/src/libraries/InterestRate.sol#L153-L171)

**Recommendation**

There is no need to cache the interest rate for a vault as it should always be calculated dynamically from the IRM. The only thing that needs to be updated is `s.endRateAt` and `s.lastUpdate` as is done in `AdaptiveIRM#updateInterestRateAtTarget`.

**Remediation**

Structurally fixed in [fddfdec](https://github.com/hyperstable/contracts/commit/fddfdec369d12b00520f98d88310a177a8c87da6). Although debt and collateral are always 0 in changes, additional changes in next edition of contracts will utilize proper values.

---
### Example 3

**Auto Label:** Inaccurate interest rate or debt calculation due to flawed state updates, race conditions, or incorrect math, leading to erroneous profit/loss, inflated debt, or unintended interest accrual.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-02-yieldoor-judging/issues/190 

## Found by 
future2\_22

### Summary
The decimal precision of the collateral is insufficient for calculating the interest, especially for wbtc.

### Root Cause
In `ReserveLogic::_updateIndexes::L156`, the totalBorrows may not update or may experience precision loss.
https://github.com/sherlock-audit/2025-02-yieldoor/blob/main/yieldoor/src/libraries/ReserveLogic.sol#L156
```solidity
156:        newTotalBorrows = newBorrowingIndex * (reserve.totalBorrows) / (reserve.borrowingIndex);
```
To change the `totalBorrows`, the unaccounted interest must be equal to or greater than 1 wei. 
To avoid precision loss, the interest must exceed 10,000 wei. 
If the collateral is wbtc, this value is not small.
And the `_updateIndexes` function could be executed every block(period = 15s), and the unaccounted interest may be less than the above value.

### Internal pre-conditions
N/A

### External pre-conditions
N/A

### Attack Path
An attacker can repeatedly deposit 1001 wei of wbtc and redeem the maximum amount of wbtc (type(uint256).max). 
Alternatively, users can interact with the `LendingPool` frequently

### PoC
Let's consider the following scenario.
LendingPool's asset = wbtc, BorrowingRateConfig: (0%, 0%) -> (80%, 20%) -> (90%, 50%) -> (100%, 150%)
totalLiquidityAndBorrows = 0.25 wbtc, totalBorrows = 0.126144 wbtc
currentUtilizationRate = 50.4576%, 
https://github.com/sherlock-audit/2025-02-yieldoor/blob/main/yieldoor/src/libraries/InterestRateUtils.sol#L28
currentBorrowingRate = 50.4576% * 20% / 80% = 12.6144%
https://github.com/sherlock-audit/2025-02-yieldoor/blob/main/yieldoor/src/libraries/InterestRateUtils.sol#L73
rate = 12.6144% = 0.126144e27, currentTimestamp = lastUpdateTimestamp + 15.
ratePerSecond = rate / SECONDS_PER_YEAR = 4e18
basePowerTwo = ratePerSecond * ratePerSecond / PRECISION = 16e9
basePowerThree = 64
secondTerm = exp * expMinusOne * basePowerTwo / 2 = 15 * 14 * 16e9 = 3360e9
thirdTerm = exp * expMinusOne * expMinusTwo * basePowerThree / 6 = 15 * 14 * 13 * 64 / 6 = 29120
CompoundedInterest = (PRECISION) + (ratePerSecond * exp) + (secondTerm) + (thirdTerm) = 
    = 1e27 + 4e18 * 15 + 3360e9 + 29120 = 1e27 + 60e18 + 3360e9 + 29120 < 1e27 + 60.1e18
https://github.com/sherlock-audit/2025-02-yieldoor/blob/main/yieldoor/src/libraries/ReserveLogic.sol#L115
latestBorrowingIndex = reserve.borrowingIndex * calculateCompoundedInterest / 1e27 < reserve.borrowingIndex * (1e27 + 60.1e18) / 1e27
https://github.com/sherlock-audit/2025-02-yieldoor/blob/main/yieldoor/src/libraries/ReserveLogic.sol#L156
newTotalBorrows = latestBorrowingIndex * (reserve.totalBorrows) / (reserve.borrowingIndex) < 
    < (reserve.borrowingIndex * (1e27 + 60.1e18) / 1e27) * (reserve.totalBorrows) / (reserve.borrowingIndex) = 
    = (1e27 + 60.1e18) / 1e27 * (reserve.totalBorrows) = 
    = reserve.totalBorrows + 60.1e-9 * 0.126144e8 < 
    < reserve.totalBorrows + 1;
Therefore, newTotalBorrows = reserve.totalBorrows

### Impact
The depositor of the LendingPool may not be able to take any profits at all or may receive reduced profits.

When `BorrowingRateConfig := (0%, 0%) -> (80%, 20%) -> (90%, 50%) -> (100%, 150%)`,
1. LendingPool's asset = wbtc, totalLiquidityAndBorrows = 0.25 wbtc, totalBorrows = 0.126144 wbtc
If the `_updateIndexes` function is executed every block, the interest is not collected. 
If it is executed every 100 blocks (25 mins), the calculation of interest experiences 1% precision loss.

2. LendingPool's asset = wbtc, totalLiquidityAndBorrows = 25 wbtc, totalBorrows = 12.6144 wbtc
If the `_updateIndexes` function is excuted every 100 block (25 mins), the calculation of interest experiences 0.01% precision loss.

3. LendingPool's asset = usdc, totalLiquidityAndBorrows = 250,000 usdc, totalBorrows = 126,144 usdc
If the `_updateIndexes` function is excuted every block, the calculation of interest experiences 0.01% precision loss.

### Mitigation
Consider increasing the decimal precision of the collateral.

---
