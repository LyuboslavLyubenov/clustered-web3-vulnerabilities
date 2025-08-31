# Cluster -1449

**Rank:** #343  
**Count:** 19  

## Label
Missing zero-amount validation in transfers leads to reverts on tokens that reject zero-value transfers, causing denial-of-service, failed operations, or unintended state changes.

## Cluster Information
- **Total Findings:** 19

## Examples

### Example 1

**Auto Label:** Improper zero-value transfer handling leads to reverts, denial-of-service, or incorrect state reporting, compromising transaction integrity and system reliability.  

**Original Text Preview:**

Inside `executeManualNegativePnlRealizationCallback`, it calculates `realizedNegativePnlToCancelCollateral` and attempts to transfer the assets from the vault to the `GNSMultiCollatDiamond` contract.

```solidity
    function executeManualNegativePnlRealizationCallback(
        ITradingStorage.PendingOrder memory _order,
        ITradingCallbacks.AggregatorAnswer memory _a
    ) external {
        ITradingStorage.Trade memory trade = _getTrade(_order.trade.user, _order.trade.index);

        if (trade.isOpen) {
            int256 unrealizedPnlCollateral = TradingCommonUtils.getTradeUnrealizedRawPnlCollateral(
                trade,
                _a.current
            );
            uint256 unrealizedNegativePnlCollateral = unrealizedPnlCollateral < 0
                ? uint256(-unrealizedPnlCollateral)
                : uint256(0);

            uint256 existingManuallyRealizedNegativePnlCollateral = _getMultiCollatDiamond()
                .getTradeManuallyRealizedNegativePnlCollateral(trade.user, trade.index);
            uint256 newManuallyRealizedNegativePnlCollateral = existingManuallyRealizedNegativePnlCollateral;

            if (unrealizedNegativePnlCollateral > existingManuallyRealizedNegativePnlCollateral) {
                // ...

                newManuallyRealizedNegativePnlCollateral += negativePnlToRealizeCollateral;
            } else {
>>>             uint256 realizedNegativePnlToCancelCollateral = existingManuallyRealizedNegativePnlCollateral -
                    unrealizedNegativePnlCollateral;

>>>             TradingCommonUtils.receiveCollateralFromVault(
                    trade.collateralIndex,
                    realizedNegativePnlToCancelCollateral
                );

                newManuallyRealizedNegativePnlCollateral -= realizedNegativePnlToCancelCollateral;
            }

            _getMultiCollatDiamond().storeManuallyRealizedNegativePnlCollateral(
                trade.user,
                trade.index,
                newManuallyRealizedNegativePnlCollateral
            );

            emit ITradingCallbacksUtils.TradeNegativePnlManuallyRealized(
                _a.orderId,
                trade.collateralIndex,
                trade.user,
                trade.index,
                unrealizedNegativePnlCollateral,
                existingManuallyRealizedNegativePnlCollateral,
                newManuallyRealizedNegativePnlCollateral,
                _a.current
            );
        }

        _getMultiCollatDiamond().closePendingOrder(_a.orderId);
    }
```

However, it doesn't account for scenarios where `realizedNegativePnlToCancelCollateral` is 0. If the collateral transfer reverts on a 0 transfer, the operation will fail. Only trigger `receiveCollateralFromVault` when `realizedNegativePnlToCancelCollateral` is greater than 0.

---
### Example 2

**Auto Label:** Missing zero-amount validation in transfers leads to reverts on tokens that reject zero-value transfers, causing denial-of-service, failed operations, or unintended state changes.  

**Original Text Preview:**

**Description:** Per the ERC-4626 specification, the preview functions "MUST NOT revert due to vault specific user/global limits. MAY revert due to other conditions that would also cause mint/deposit/redeem/withdraw to revert".

```solidity
    function totalAccruedUSDe() public view returns (uint256) {
@>      uint pUSDeAssets = super.totalAssets();  // @audit - should return early if pUSDeAssets is zero to avoid reverting in the call below

@>      uint USDeAssets = _convertAssetsToUSDe(pUSDeAssets, true);
        return USDeAssets;
    }

    function _convertAssetsToUSDe (uint pUSDeAssets, bool withYield) internal view returns (uint256) {
@>      uint sUSDeAssets = pUSDeVault.previewRedeem(withYield ? address(this) : address(0), pUSDeAssets); // @audit - this can revert if passing yUSDe as the caller when it has no pUSDe balance
        uint USDeAssets = sUSDe.previewRedeem(sUSDeAssets);
        return USDeAssets;
    }

    function previewDeposit(uint256 pUSDeAssets) public view override returns (uint256) {
        uint underlyingUSDe = _convertAssetsToUSDe(pUSDeAssets, false);

@>      uint yUSDeShares = _valueMulDiv(underlyingUSDe, totalAssets(), totalAccruedUSDe(), Math.Rounding.Floor); // @audit - should explicitly handle the case where totalAccruedUSDe() returns zero rather than relying on _valueMulDiv() behaviour
        return yUSDeShares;
    }

    function previewMint(uint256 yUSDeShares) public view override returns (uint256) {
@>      uint underlyingUSDe = _valueMulDiv(yUSDeShares, totalAccruedUSDe(), totalAssets(), Math.Rounding.Ceil); // @audit - should explicitly handle the case where totalAccruedUSDe() and/or totalAssets() returns zero rather than relying on _valueMulDiv() behaviour
        uint pUSDeAssets = pUSDeVault.previewDeposit(underlyingUSDe);
        return pUSDeAssets;
    }

    function _valueMulDiv(uint256 value, uint256 mulValue, uint256 divValue, Math.Rounding rounding) internal view virtual returns (uint256) {
        return value.mulDiv(mulValue + 1, divValue + 1, rounding);
    }
```

As noted using `// @audit` tags in the code snippets above, `yUSDeVault::previewMint` and `yUSDeVault::previewDeposit` can revert for multiple reasons, including:
* when the pUSDe balance of the yUSDe vault is zero.
* when `pUSDeVault::previewRedeem` reverts due to division by zero in `pUSDeVault::previewYield`, invoked from `_convertAssetsToUSDe()` within `totalAccruedUSDe()`.

```solidity
     function previewYield(address caller, uint256 shares) public view virtual returns (uint256) {
        if (PreDepositPhase.YieldPhase == currentPhase && caller == address(yUSDe)) {

            uint total_sUSDe = sUSDe.balanceOf(address(this));
            uint total_USDe = sUSDe.previewRedeem(total_sUSDe);

            uint total_yield_USDe = total_USDe - Math.min(total_USDe, depositedBase);

@>          uint y_pUSDeShares = balanceOf(caller); // @audit - should return early if this is zero to avoid reverting below
@>          uint caller_yield_USDe = total_yield_USDe.mulDiv(shares, y_pUSDeShares, Math.Rounding.Floor);

            return caller_yield_USDe;
        }
        return 0;
    }

    function previewRedeem(address caller, uint256 shares) public view virtual returns (uint256) {
        return previewRedeem(shares) + previewYield(caller, shares);
    }
```

While a subset of these reverts could be considered "due to other conditions that would also cause deposit to revert", such as due to overflow, it would be better to explicitly handle these other edge cases. Additionally, even when called in isolation `yUSDeVault::totalAccruedUSDe` will revert if the pUSDe balance of the yUSDeVault is zero. Instead, this should simply return zero.

**Strata:** Fixed in commit [0f366e1](https://github.com/Strata-Money/contracts/commit/0f366e192941c875b651ee4db89b9fd3242a5ac0).

**Cyfrin:** Verified. The zero assets/shares edge cases are now explicitly handled in `yUSDeVault::_convertAssetsToUSDe` and pUSDeVault::previewYield`, including when the `yUSDe` state is not initialized as so will be equal to the zero address.

\clearpage

---
### Example 3

**Auto Label:** Missing zero-amount validation in transfers leads to reverts on tokens that reject zero-value transfers, causing denial-of-service, failed operations, or unintended state changes.  

**Original Text Preview:**

**Impact**
Certain tokens revert on a 0 amount transfer

The code also performs the check to prevent zero transfers in many parts, but in these 2 instances the check was missed

[Note that for OZ tokens, the check is not necessary](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/c304b6710b4b5fcf2a319ad28c36c49df6caef14/contracts/token/ERC20/ERC20.sol#L183-L211)

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/SwapPair.sol#L234-L238

```solidity
    if (amount0InFee > 0) {
      uint amount0GovFee = (amount0InFee * swapOperations.getGovSwapFee()) / DECIMAL_PRECISION;
      _safeTransfer(token0, tokenManager.govPayoutAddress(), amount0GovFee); /// @audit 0 revert
      balance0 -= amount0GovFee;
    } 
```

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/SwapPair.sol#L161-L163

```solidity
    // payout whats left
    _safeTransfer(token0, to, amount0 - burned0);
    _safeTransfer(token1, to, amount1 - burned1);
```

**Mitigation**

Check that each transfer is done only on non-zero amounts

---
