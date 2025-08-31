# Cluster -1035

**Rank:** #203  
**Count:** 54  

## Label
Insufficient balance validation and improper state tracking lead to unauthorized fund manipulation, incorrect repayment recognition, and loss of assets due to timing flaws and token dynamics.

## Cluster Information
- **Total Findings:** 54

## Examples

### Example 1

**Auto Label:** Failure to validate asset balances before initiating withdrawals, leading to reverts due to zero-amount validation checks in lending protocols.  

**Original Text Preview:**

**Severity:** Low

**Path:** src/stHYPEWithdrawalModule.sol#L312

**Description:** [AAVE V3 Pool](https://github.com/aave/aave-v3-core/blob/782f51917056a53a2c228701058a6c3fb233684a/contracts/protocol/libraries/logic/ValidationLogic.sol#L66-L71) reverts when withdrawing zero amount:
```
function validateWithdraw(
    DataTypes.ReserveCache memory reserveCache,
    uint256 amount,
    uint256 userBalance
  ) internal view {
    require(amount != 0, Errors.INVALID_AMOUNT);
```
Thus, when updating a lending module (`AaveLendingModule.sol`) with `assetBalance() = 0` via `stHYPEWithdrawalModule::setProposedLendingModule` it will revert:
```
if (address(lendingModule) != address(0)) {
    lendingModule.withdraw(lendingModule.assetBalance(), address(this));
}
```
```
function setProposedLendingModule() external onlyOwner {
    if (lendingModuleProposal.startTimestamp > block.timestamp) {
        revert stHYPEWithdrawalModule__setProposedLendingModule_ProposalNotActive();
    }

    if (lendingModuleProposal.startTimestamp == 0) {
        revert stHYPEWithdrawalModule__setProposedLendingModule_InactiveProposal();
    }

    if (address(lendingModule) != address(0)) {
        lendingModule.withdraw(lendingModule.assetBalance(), address(this));
    }

    lendingModule = ILendingModule(lendingModuleProposal.lendingModule);
    delete lendingModuleProposal;
    emit LendingModuleSet(address(lendingModule));
}
```

**Remediation:**  Consider checking the `assetBalance()` to not being 0 before calling `lendingModule.withdraw()` inside `stHYPEWithdrawalModule::setProposedLendingModule`.

For example:
```
if (address(lendingModule) != address(0) && lendingModule.assetBalance() != 0) {
    lendingModule.withdraw(lendingModule.assetBalance(), address(this));
}
```

**Status:** Fixed


- - -

---
### Example 2

**Auto Label:** **Incorrect state updates due to flawed logic and missing boundaries, leading to erroneous balances, locked assets, and compromised protocol stability.**  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

- In `processWithdrawals()` function: when the `_withdrawShares = 0`, the `s.totalLiquidity` is increased by the liquidity of the collected-and-deposited fees of the position, where it assumes that the totalLiquidity of the position will not be decreased when `_reviseHedge()` is called:

```javascript
 function processWithdrawals(
        bytes memory path0,
        bytes memory path1
    ) external virtual override {
        //...
        uint256 _liquidity = _depositFees(_tokenId, path0, path1);
        _totalLiquidity += _liquidity;

        if (_withdrawShares == 0) {
            // if no withdrawals we only compound fees
            if (_liquidity > 0) {
                _reviseHedge(_totalLiquidity, path0, path1); // liquidity only goes up here
            }
             s.totalLiquidity = _totalLiquidity;
            FundingVault(s.withdrawVault).disburseClaims();
            return;
        }

        //...
    }
```

Where:

```javascript
 function _reviseHedge(
        uint256 _totalLiquidity,
        bytes memory path0,
        bytes memory path1
    ) internal virtual {
        uint256 _newHedgeLiquidity = (_totalLiquidity * s.hedgeSize) / 1e18;

        uint256 _prevHedgeLiquidity = _loanData.liquidity;

        if (_newHedgeLiquidity > _prevHedgeLiquidity) {
            _increaseHedge(
                _newHedgeLiquidity - _prevHedgeLiquidity,
                path0,
                path1
            );
        } else if (_newHedgeLiquidity < _prevHedgeLiquidity) {
            _decreaseHedge(
                _prevHedgeLiquidity - _newHedgeLiquidity,
                path0,
                path1
            );
        }
    }
```

- Given that there's a setter for the `hedgeSize` (where it's used to calculate the `_newHedgeLiquidity`), then the `s.totalLiquidity` should be updated based on the liquidity position in case the `hedgeSize` was updated (also, in `_reviseHedge()`, the hedge liquidity is fetched based on the last updated data without checking if the hedge has accrued interest over time).

- So if the new `hedgeSize` is decreased and results in `_newHedgeLiquidity < _prevHedgeLiquidity` , then the hedge will be decreased, which might result in the hedge incurring losses, hence affecting the liquidity, which will affect the next process that's going to be executed after the withdrawal: if the next process is a deposit, then this incorrect `s.totalLiquidity` will result in minting wrong shares for that deposited period as the liquidity doesn't reflect the actual one (since it's the cashed one + liquidity from the collected-and-deposited fees of the LP position).

- Same issue in `DepositProcess.processDeposits()` function.

## Recommendations

```diff
 function processWithdrawals(
        bytes memory path0,
        bytes memory path1
    ) external virtual override {
        //...
        uint256 _liquidity = _depositFees(_tokenId, path0, path1);
        _totalLiquidity += _liquidity;

        if (_withdrawShares == 0) {
            // if no withdrawals we only compound fees
            if (_liquidity > 0) {
                _reviseHedge(_totalLiquidity, path0, path1); // liquidity only goes up here
            }
-            s.totalLiquidity = _totalLiquidity;
+            s.totalLiquidity = _getTotalLiquidity(s.tokenId);
            FundingVault(s.withdrawVault).disburseClaims();
            return;
        }

        //...
    }
```

---
### Example 3

**Auto Label:** Flawed balance validation and mismanagement of gas/funds due to incorrect state deductions, leading to lost funds, false reverts, and inconsistent accounting.  

**Original Text Preview:**

## Severity: Low Risk

## Context 
- GasAccounting.sol#L143-L202
- GasAccounting.sol#L254-L298

## Description
Function `_assign()` updates deposits, but the mirror function `_credit()` doesn't update withdrawals. Since it can be called from multiple locations, this is particularly important. `_credit()` can only be called via `_settle()`. However, after this call, there is an external call via `safeTransferETH()`, making it potentially risky not to update withdrawals. See issue "Call to safeTransferETH can do unwanted actions". As far as we can see, no harm can be done.

```solidity
function _settle( /*...*/ ) /*...*/ {
    // ...
    if (_deposits < _claims + _withdrawals) {
        // ...
        if (_assign(winningSolver, amountOwed, true, false)) {
            revert InsufficientTotalBalance((_claims + _withdrawals) - deposits); // uses updated deposits , !
        }
    } else {
        // ...
        _credit(winningSolver, amountCredited);
    }
    // ...
    SafeTransferLib.safeTransferETH(bundler, _claims);
    // ...
}

function _assign(address owner, uint256 amount, bool solverWon, bool bidFind) internal returns (bool isDeficit) { 
    // ...
    bondedTotalSupply -= amount;
    deposits += amount;
}

function _credit(address owner, uint256 amount) internal {
    // ...
    bondedTotalSupply += amount;
    // ... // no change in withdrawals
}
```

## Recommendation
Double-check there are no side effects from not increasing withdrawals. Consider adding a comment in function `_credit()` and/or `_assign()`.

## Fastlane
- Solved in PR 246.

## Spearbit
- Verified.

---
