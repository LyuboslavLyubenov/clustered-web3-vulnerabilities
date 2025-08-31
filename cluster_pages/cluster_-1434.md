# Cluster -1434

**Rank:** #88  
**Count:** 196  

## Label
Failure to account for accrued interest in rate calculations and state updates leads to mispriced rates, retroactive rate application, and inaccurate utilization, enabling financial exploitation and protocol instability.

## Cluster Information
- **Total Findings:** 196

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

**Auto Label:** Misaligned cross-chain data filtering and state referencing cause incorrect collateral, interest, and reward calculations, leading to flawed debt exposure and failed liquidations.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/1009 

## Found by 
0xc0ffEE, JeRRy0422, future, jokr, newspacexyz, t.aksoy

### Summary

Using the borrow index of LToken on the same chain for cross chain borrow can cause the cross chain borrow to be incorrectly accrued interest

### Root Cause

The function [`LendStorage::borrowWithInterest()` ](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/LendStorage.sol#L478-L503) is used to calculate cross borrow with interest. However, the parameters `_lToken` actually the LToken on the same chain. The function is accruing cross chain borrow interest **with same-chain LToken borrow index**. This can cause debt calculation of cross chain borrows to be imprecise, hence impacting cross chain functionality.
```solidity
    function borrowWithInterest(address borrower, address _lToken) public view returns (uint256) {
        address _token = lTokenToUnderlying[_lToken];
        uint256 borrowedAmount;

        Borrow[] memory borrows = crossChainBorrows[borrower][_token];
        Borrow[] memory collaterals = crossChainCollaterals[borrower][_token];

        require(borrows.length == 0 || collaterals.length == 0, "Invariant violated: both mappings populated");
        // Only one mapping should be populated:
        if (borrows.length > 0) {
            for (uint256 i = 0; i < borrows.length; i++) {
                if (borrows[i].srcEid == currentEid) {
                    borrowedAmount +=
@>                        (borrows[i].principle * LTokenInterface(_lToken).borrowIndex()) / borrows[i].borrowIndex;
                }
            }
        } else {
            for (uint256 i = 0; i < collaterals.length; i++) {
                // Only include a cross-chain collateral borrow if it originated locally.
                if (collaterals[i].destEid == currentEid && collaterals[i].srcEid == currentEid) {
                    borrowedAmount +=
@>                        (collaterals[i].principle * LTokenInterface(_lToken).borrowIndex()) / collaterals[i].borrowIndex;
                }
            }
        }
        return borrowedAmount;
    }
```

### Internal Pre-conditions

NA

### External Pre-conditions

NA

### Attack Path

NA

### Impact

- Cross chain borrow accrues interest incorrectly
- Cross chain functionalities does not work properly 

### PoC

_No response_

### Mitigation

Consider changing the mechanism to accrue cross chain borrow index

---
### Example 3

**Auto Label:** Misaligned cross-chain data filtering and state referencing cause incorrect collateral, interest, and reward calculations, leading to flawed debt exposure and failed liquidations.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/946 

## Found by 
0x1422, 0xB4nkz, 0xubermensch, 0xzey, Allen\_George08, Brene, JeRRy0422, Kvar, Lamsya, TopStar, Ziusz, algiz, anchabadze, dimah7, dystopia, freeking, gkrastenov, heavyw8t, ivanalexandur, m3dython, mahdifa, oxelmiguel, t.aksoy, x0rc1ph3r, xiaoming90, zxriptor

### Summary

The `borrowWithInterest`  function can not correctly calculate the cross-chain collaterals, which affects the entire repayment logic.

### Root Cause

When a user tries to make a cross-chain borrow from source chain A to destination chain B, their `crossChainBorrows` are stored on chain B, and their `crossChainCollaterals `are stored on chain A. The cross-chain collateral is stored during the `_handleValidBorrowRequest` function:

```solidity
   lendStorage.addCrossChainCollateral(
                payload.sender, // user
                destUnderlying,  // underlying
                LendStorage.Borrow({
                   //@audit-issue srcEid != destEid
                    srcEid: srcEid,
                    destEid: currentEid,
                    principle: payload.amount,
                    borrowIndex: currentBorrowIndex,
                    borrowedlToken: payload.destlToken,
                    srcToken: payload.srcToken
                })
            );
```

When the user tries to repay their borrow, the system checks the sum of the cross-chain collaterals on either side. In the `borrowWithInterest` function, while calculating the total collateral, it checks whether:

https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/LendStorage.sol#L497

```solidity
 if (collaterals[i].destEid == currentEid && collaterals[i].srcEid == currentEid)
```

This condition is always false, because `destEid` is always different from `srcEid`. As a result, the amount stored for cross-chain collaterals is never included in the borrow amount calculation, potentially leading to incorrect or failed repayments.

### Internal Pre-conditions

N/A

### External Pre-conditions

User should have cross-chain borrow.

### Attack Path

N/A, just need to be called `repayCrossChainBorrow` function.

### Impact

An incorrect calculation of the borrowed amount will be made as the cross-chain collateral is not included. This can potentially lead to incorrect or failed repayments.

### PoC

_No response_

### Mitigation

Avoid checking if `collaterals[i].destEid == currentEid && collaterals[i].srcEid == currentEid` is true. This condition should be rewritten.

---
