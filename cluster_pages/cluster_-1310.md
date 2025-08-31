# Cluster -1310

**Rank:** #256  
**Count:** 35  

## Label
Missing safety buffers and validation checks in liquidation logic lead to unjustified, cascading, or exploitable liquidations, exposing users and lenders to financial loss and systemic risk.

## Cluster Information
- **Total Findings:** 35

## Examples

### Example 1

**Auto Label:** **Incentive misalignment in liquidation mechanisms leads to unprofitable or harmful liquidations, enabling bad debt accumulation and systemic risk.**  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/660 

## Found by 
0xpetern, TessKimy, ifeco445, rudhra1749, theweb3mechanic

### Summary

https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/713372a1ccd8090ead836ca6b1acf92e97de4679/Lend-V2/src/LayerZero/CoreRouter.sol#L230-L245
```solidity
    function liquidateBorrow(address borrower, uint256 repayAmount, address lTokenCollateral, address borrowedAsset)
        external
    {
        // The lToken of the borrowed asset
        address borrowedlToken = lendStorage.underlyingTolToken(borrowedAsset);


        LTokenInterface(borrowedlToken).accrueInterest();


        (uint256 borrowed, uint256 collateral) =
            lendStorage.getHypotheticalAccountLiquidityCollateral(borrower, LToken(payable(borrowedlToken)), 0, 0);


        liquidateBorrowInternal(
            msg.sender, borrower, repayAmount, lTokenCollateral, payable(borrowedlToken), collateral, borrowed
        );
    }


```
protocol allows for partial liquidation, if a borrowers position is insolvent then a liquidator can liquidate him leaving bad debt to protocol.And there is no bad debt accounting mechanism in protocol, due to which protocol users should bare this bad debt loss 


### Root Cause

no mechanism to account for bad debt .

### Internal Pre-conditions

none 

### External Pre-conditions

none

### Attack Path

liquidator liquidates a undercollaterized insolvent borrow position due to which bad debt accures.

### Impact

as there is no mechanism to account for bad debt of protocol this loss will be effected to remaining protocol users 

### PoC

_No response_

### Mitigation

implement a mechanism for bad debt accounting

---
### Example 2

**Auto Label:** **Incentive misalignment in liquidation mechanisms leads to unprofitable or harmful liquidations, enabling bad debt accumulation and systemic risk.**  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/636 

## Found by 
0x23r0, 0xc0ffEE, Kvar, MRXSNOWDEN, Phaethon, SarveshLimaye, Tychai0s, alphacipher, anchabadze, dmdg321, dystopia, future, ggg\_ttt\_hhh, hgrano, jokr, oxelmiguel, stonejiajia, t.aksoy

### Summary
In the current implementation of cross-chain liquidation, when a liquidation is requested on chainB (the destination chain), there is no transfer of the underlying token from the liquidator. If the seize operation succeeds on chainA (the source chain), the underlying tokens are transferred to the liquidator, and the repayment is executed on chainB. However, if the seize fails, the underlying tokens are transferred from the contract to the liquidator on chainB.

In this scenario, if the seize operation is successful but the liquidator has not approved the underlying token on chainB, the repayment may fail while the liquidator still successfully receives the underlying tokens from the liquidatee on chainA. This creates a potential exploit.

### Root Cause
The root cause of this issue is that, on chainB, there is no transfer of the underlying tokens from the liquidator at the start of the liquidation process.

### Internal pre-conditions
N/A

### External pre-conditions
N/A

### Attack Path
N/A

### PoC
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L172-L192
```solidity
    function liquidateCrossChain(
        ...
    ) external {
        LendStorage.LiquidationParams memory params = LendStorage.LiquidationParams({
            ...
        });

        _validateAndPrepareLiquidation(params);
        _executeLiquidation(params);
    }
```
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L197-L233
```solidity
    function _validateAndPrepareLiquidation(LendStorage.LiquidationParams memory params) private view {
        require(params.borrower != msg.sender, "Liquidator cannot be borrower");
        require(params.repayAmount > 0, "Repay amount cannot be zero");

        // Get the lToken for the borrowed asset on this chain
        params.borrowedlToken = lendStorage.underlyingTolToken(params.borrowedAsset);
        require(params.borrowedlToken != address(0), "Invalid borrowed asset");

        // Important: Use underlying token addresses consistently
        address borrowedUnderlying = lendStorage.lTokenToUnderlying(params.borrowedlToken);

        // Verify the borrow position exists and get details
        LendStorage.Borrow[] memory userCrossChainCollaterals =
            lendStorage.getCrossChainCollaterals(params.borrower, borrowedUnderlying);
        bool found = false;

        for (uint256 i = 0; i < userCrossChainCollaterals.length;) {
            if (userCrossChainCollaterals[i].srcEid == params.srcEid) {
                found = true;
                params.storedBorrowIndex = userCrossChainCollaterals[i].borrowIndex;
                params.borrowPrinciple = userCrossChainCollaterals[i].principle;
                break;
            }
            unchecked {
                ++i;
            }
        }
        require(found, "No matching borrow position");

        // Validate liquidation amount against close factor
        uint256 maxLiquidationAmount = lendStorage.getMaxLiquidationRepayAmount(
            params.borrower,
            params.borrowedlToken,
            false // cross-chain liquidation
        );
        require(params.repayAmount <= maxLiquidationAmount, "Exceeds max liquidation");
    }
```
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L235-L243
```solidity
    function _executeLiquidation(LendStorage.LiquidationParams memory params) private {
        // First part: Validate and prepare liquidation parameters
        uint256 maxLiquidation = _prepareLiquidationValues(params);

        require(params.repayAmount <= maxLiquidation, "Exceeds max liquidation");

        // Secon part: Validate collateral and execute liquidation
        _executeLiquidationCore(params);
    }
```
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L264-L285
```solidity
    function _executeLiquidationCore(LendStorage.LiquidationParams memory params) private {
        // Calculate seize tokens
        address borrowedlToken = lendStorage.underlyingTolToken(params.borrowedAsset);

        (uint256 amountSeizeError, uint256 seizeTokens) = LendtrollerInterfaceV2(lendtroller)
            .liquidateCalculateSeizeTokens(borrowedlToken, params.lTokenToSeize, params.repayAmount);

        require(amountSeizeError == 0, "Seize calculation failed");

        // Send message to Chain A to execute the seize
        _send(
            params.srcEid,
            seizeTokens,
            params.storedBorrowIndex,
            0,
            params.borrower,
            lendStorage.crossChainLTokenMap(params.lTokenToSeize, params.srcEid), // Convert to Chain A version before sending
            msg.sender,
            params.borrowedAsset,
            ContractType.CrossChainLiquidationExecute
        );
    }
```

### Impact
A malicious liquidator can liquidate without providing the necessary collateral resulting the liquidatee's debt is not deducted.

### Mitigation
Consider implementing an escrow mechanism for the collateral from the liquidator before initiating the liquidation process to ensure that the collateral is secured.

---
### Example 3

**Auto Label:** **Incentive misalignment in liquidation mechanisms leads to unprofitable or harmful liquidations, enabling bad debt accumulation and systemic risk.**  

**Original Text Preview:**

The documentation states that the liquidator receives a collateral reward equal to 105% of the liquidated asset value. However, in the contract, the liquidation reward is not fixed at 105% but instead depends on the vault's MCR and `liquidatorRewardBps`.

```solidity
        values.debtToRepay = FixedPointMathLib.min(_debtToRepay, _positionData.debtSnapshot);
        uint256 equivalentCollateral = values.debtToRepay.divWad(values.sharePrice);
        uint256 overcollateralization = equivalentCollateral.mulWad(_vaultData.MCR - 1e18);
        uint256 requiredCollateral = equivalentCollateral + overcollateralization.mulWad(liquidatorRewardBps);
```

It is recommended to either update the documentation or adjust the contract logic to ensure consistency between them.

---
