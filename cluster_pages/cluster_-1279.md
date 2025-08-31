# Cluster -1279

**Rank:** #258  
**Count:** 34  

## Label
Failure to properly update or validate state during critical operations leads to inaccurate execution prices, corrupted state, or incorrect rewards—enabling exploitable profit opportunities or financial misrepresentation.

## Cluster Information
- **Total Findings:** 34

## Examples

### Example 1

**Auto Label:** Failure to properly update or validate state during critical operations leads to inaccurate execution prices, corrupted state, or incorrect rewards—enabling exploitable profit opportunities or financial misrepresentation.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** High

## Description

Upon proposing a token swap, we have the following sequence of function calls:

```solidity
uint256[] memory totalValues = new uint256[](numBaskets);
// 2d array of asset balances for each basket
uint256[][] memory basketBalances = new uint256[][](numBaskets);
_initializeBasketData(self, baskets, basketAssets, basketBalances, totalValues);
// NOTE: for rebalance retries the internal trades must be updated as well
_processInternalTrades(self, internalTrades, baskets, basketBalances);
_validateExternalTrades(self, externalTrades, baskets, totalValues, basketBalances);
if (!_isTargetWeightMet(self, baskets, basketTargetWeights, basketAssets, basketBalances, totalValues)) {
         revert TargetWeightsNotMet();
}
```

The `totalValues` array is the total USD value of a basket, a basket per element of the array. It is populated upon calling `_initialBasketData()`, then it is provided in `_validateExternalTrades()` where the array is manipulated based on the trades. Afterwards, it is used upon checking the deviation in `_isTargetWeightMet()`:

```solidity
uint256 afterTradeWeight = FixedPointMathLib.fullMulDiv(assetValueInUSD, _WEIGHT_PRECISION, totalValues[i]);
if (MathUtils.diff(proposedTargetWeights[j], afterTradeWeight) > _MAX_WEIGHT_DEVIATION) {
         return false;
}
```

The code functions correctly assuming that the USD value of a basket stays stationary during the `_processInternalTrades()` during the call. However, that is not actually the case due to the fact that there is a fee upon processing the internal trades:

```solidity
if (swapFee > 0) {
                info.feeOnSell = FixedPointMathLib.fullMulDiv(trade.sellAmount, swapFee, 20_000);
                self.collectedSwapFees[trade.sellToken] += info.feeOnSell;
                emit SwapFeeCharged(trade.sellToken, info.feeOnSell);
            }
if (swapFee > 0) {
                info.feeOnBuy = FixedPointMathLib.fullMulDiv(initialBuyAmount, swapFee, 20_000);
                self.collectedSwapFees[trade.buyToken] += info.feeOnBuy;
                emit SwapFeeCharged(trade.buyToken, info.feeOnBuy);
            }
```

This results in the USD value of both the `fromBasket` and the `toBasket` going down based on the fee amount, thus the deviation weight check will be inaccurate as the `totalValues` array is not changed during the internal trades processing, it is out-of-sync.

## Recommendations

Provide the total values array upon processing the internal trades and account for the fees applied.

---
### Example 2

**Auto Label:** Failure to properly update or validate state during critical operations leads to inaccurate execution prices, corrupted state, or incorrect rewards—enabling exploitable profit opportunities or financial misrepresentation.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/773 

## Found by 
0xc0ffEE, Kirkeelee, Kvar, PNS, future, ggg\_ttt\_hhh, hard1k, jokr, mussucal, patitonar, rudhra1749, t.aksoy

### Summary

if we observe  _handleBorrowCrossChainRequest function,
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L581-L660
```solidity
    function _handleBorrowCrossChainRequest(LZPayload memory payload, uint32 srcEid) private {
        // Accrue interest on borrowed token on destination chain
        LTokenInterface(payload.destlToken).accrueInterest();


        // Get current borrow index from destination lToken
        uint256 currentBorrowIndex = LTokenInterface(payload.destlToken).borrowIndex();


        // Important: Use the underlying token address
        address destUnderlying = lendStorage.lTokenToUnderlying(payload.destlToken);


        // Check if user has any existing borrows on this chain
        bool found = false;
        uint256 index;


        LendStorage.Borrow[] memory userCrossChainCollaterals =
            lendStorage.getCrossChainCollaterals(payload.sender, destUnderlying);


        /**
         * Filter collaterals for the given srcEid. Prevents borrowing from
         * multiple collateral sources.
         */
        for (uint256 i = 0; i < userCrossChainCollaterals.length;) {
            if (
                userCrossChainCollaterals[i].srcEid == srcEid
                    && userCrossChainCollaterals[i].srcToken == payload.srcToken
            ) {
                index = i;
                found = true;
                break;
            }
            unchecked {
                ++i;
            }
        }


        // Get existing borrow amount
        (uint256 totalBorrowed,) = lendStorage.getHypotheticalAccountLiquidityCollateral(
            payload.sender, LToken(payable(payload.destlToken)), 0, payload.amount
        );


        // Verify the collateral from source chain is sufficient for total borrowed amount
        require(payload.collateral >= totalBorrowed, "Insufficient collateral");


        // Execute the borrow on destination chain
        CoreRouter(coreRouter).borrowForCrossChain(payload.sender, payload.amount, payload.destlToken, destUnderlying);


        /**
         * @dev If existing cross-chain collateral, update it. Otherwise, add new collateral.
         */
        if (found) {
            uint256 newPrincipleWithAmount = (userCrossChainCollaterals[index].principle * currentBorrowIndex)
                / userCrossChainCollaterals[index].borrowIndex;


            userCrossChainCollaterals[index].principle = newPrincipleWithAmount + payload.amount;
            userCrossChainCollaterals[index].borrowIndex = currentBorrowIndex;


            lendStorage.updateCrossChainCollateral(
                payload.sender, destUnderlying, index, userCrossChainCollaterals[index]
            );
        } else {
            lendStorage.addCrossChainCollateral(
                payload.sender,
                destUnderlying,
                LendStorage.Borrow({
                    srcEid: srcEid,
                    destEid: currentEid,
                    principle: payload.amount,
                    borrowIndex: currentBorrowIndex,
                    borrowedlToken: payload.destlToken,
                    srcToken: payload.srcToken
                })
            );
        }


        // Track borrowed asset
        lendStorage.addUserBorrowedAsset(payload.sender, payload.destlToken);


        // Distribute LEND rewards on destination chain
        lendStorage.distributeBorrowerLend(payload.destlToken, payload.sender);


```
here if we observe, first it updates CrossChainCollaterals mapping then it calls  lendStorage.distributeBorrowerLend which gives Lend tokens to borrower.
But it should first distribute lend tokens and then it should update CrossChainCollaterals mapping.because distributeBorrowerLend function uses CrossChainCollaterals mapping to calculate total user borrowed assets,
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/LendStorage.sol#L365-L368
```solidity
        uint256 borrowerAmount = div_(
            add_(borrowWithInterest(borrower, lToken), borrowWithInterestSame(borrower, lToken)),
            Exp({mantissa: LTokenInterface(lToken).borrowIndex()})
        );
```
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/LendStorage.sol#L478-L501
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
                        (borrows[i].principle * LTokenInterface(_lToken).borrowIndex()) / borrows[i].borrowIndex;
                }
            }
        } else {
            for (uint256 i = 0; i < collaterals.length; i++) {
                // Only include a cross-chain collateral borrow if it originated locally.
                if (collaterals[i].destEid == currentEid && collaterals[i].srcEid == currentEid) {
                    borrowedAmount +=
                        (collaterals[i].principle * LTokenInterface(_lToken).borrowIndex()) / collaterals[i].borrowIndex;
                }
            }
```
due to this protocol will distrubute incorrect amount of lend tokens to users.

### Root Cause

distributing  lend tokens before updating crossChainCollateral mapping 

### Internal Pre-conditions

none

### External Pre-conditions

none

### Attack Path

none

### Impact

incorrect distribution of Lend tokens to users

### PoC

_No response_

### Mitigation

first distribute Lend tokens and then update CrossChainCollateral mapping

---
### Example 3

**Auto Label:** Inconsistent state management due to improper array indexing, missing deletions, or flawed iterator/ordering, leading to data corruption, incorrect state transitions, and erroneous operations.  

**Original Text Preview:**

## Lock Removal in SmartTable

When a lock is removed, it is not actually deleted from the SmartTable storing lock records. This renders the data accessible in the system even after it is supposedly removed. Since locks are not fully removed from the SmartTable, view functions may show locks that should have been deleted. 

Also, the `lock_index` is not validated to ensure it is within the bounds of the investor’s lock count. Thus, the `lock_index` values may be out of bounds, potentially attempting to delete nonexistent records. As a result, the same lock may be removed multiple times repeatedly, each time decreasing the lock count.

## Remediation

- Check that `lock_index <= total_investor_count` in both the view functions and the removal function.

## Patch

Resolved in 06538b8.

---
