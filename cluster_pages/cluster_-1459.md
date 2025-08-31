# Cluster -1459

**Rank:** #285  
**Count:** 27  

## Label
Inconsistent decimal handling across contracts leads to arithmetic errors, precision loss, and miscalculated token amounts, risking incorrect balances, dust accumulation, and severe financial misstatements.

## Cluster Information
- **Total Findings:** 27

## Examples

### Example 1

**Auto Label:** **Incorrect decimal handling leading to precision errors, inaccurate asset calculations, and potential financial losses due to flawed assumptions about token decimal precision.**  

**Original Text Preview:**

<https://github.com/code-423n4/2024-12-bakerfi/blob/0daf8a0547b6245faed5b6cd3f5daf44d2ea7c9a/contracts/core/strategies/StrategyLeverage.sol# L234>

<https://github.com/code-423n4/2024-12-bakerfi/blob/0daf8a0547b6245faed5b6cd3f5daf44d2ea7c9a/contracts/core/strategies/StrategyLeverage.sol# L347>

<https://github.com/code-423n4/2024-12-bakerfi/blob/0daf8a0547b6245faed5b6cd3f5daf44d2ea7c9a/contracts/core/strategies/StrategyLeverage.sol# L359>

<https://github.com/code-423n4/2024-12-bakerfi/blob/0daf8a0547b6245faed5b6cd3f5daf44d2ea7c9a/contracts/core/strategies/StrategyLeverage.sol# L673>

<https://github.com/code-423n4/2024-12-bakerfi/blob/0daf8a0547b6245faed5b6cd3f5daf44d2ea7c9a/contracts/core/strategies/StrategyLeverage.sol# L640>

<https://github.com/code-423n4/2024-12-bakerfi/blob/0daf8a0547b6245faed5b6cd3f5daf44d2ea7c9a/contracts/core/strategies/StrategySupplyBase.sol# L110>

<https://github.com/code-423n4/2024-12-bakerfi/blob/0daf8a0547b6245faed5b6cd3f5daf44d2ea7c9a/contracts/core/strategies/StrategySupplyBase.sol# L69>

### Finding description and impact

The `StrategyLeverage` contract has multiple incorrect decimal handling issues, causing the system to not support tokens with decimals other than 18.

### Proof of Concept

First, the vault contract’s share decimal is set to 18, as recommended by the ERC4626 standard. Ideally, the vault’s share decimal should reflect the underlying token’s decimal. Otherwise, conversions through `convertToShares` and `convertToAssets` would be required.

In `StrategyLeverage`, we can see that all calls to `totalAssets()` are converted to 18 decimals for share calculations.

Under the above premise, the contract has multiple decimal handling errors, making it incompatible with tokens that use decimals other than 18:

1. The `_deploy` function should return the amount in the system’s 18-decimal format, rather than the token’s native decimal format.

   
```

   function _depositInternal(uint256 assets, address receiver) private returns (uint256 shares) {
       ...
       uint256 deployedAmount = _deploy(assets);

       // Calculate shares to mint
       shares = total.toBase(deployedAmount, false);

       // Prevent inflation attack for the first deposit
       if (total.base == 0 && shares < _MINIMUM_SHARE_BALANCE) {
           revert InvalidShareBalance();
       }

       // Mint shares to the receiver
       _mint(receiver, shares);

       // Emit deposit event
       emit Deposit(msg.sender, receiver, assets, shares);
   }
   
```

   The `_deploy` function is used to calculate shares, so it should return the amount in the system’s 18-decimal format. However, the strategy always returns the amount in the token’s native decimal format.
   To address this, the `_pendingAmount` in the `_supplyBorrow` function should be converted to 18-decimal format.
2. In the `_redeemInternal` process, the `withdrawAmount` passed to `_undeploy` is in 18-decimal format (since `totalAssets` returns 18-decimal values).

   
```

   function _redeemInternal(
       uint256 shares,
       address receiver,
       address holder,
       bool shouldRedeemETH
   ) private returns (uint256 retAmount) {
       if (shares == 0) revert InvalidAmount();
       if (receiver == address(0)) revert InvalidReceiver();
       if (balanceOf(holder) < shares) revert NotEnoughBalanceToWithdraw();

       // Transfer shares to the contract if sender is not the holder
       if (msg.sender != holder) {
           if (allowance(holder, msg.sender) < shares) revert NoAllowance();
           transferFrom(holder, msg.sender, shares);
       }

       // Calculate the amount to withdraw based on shares
       uint256 withdrawAmount = (shares * totalAssets()) / totalSupply();
       if (withdrawAmount == 0) revert NoAssetsToWithdraw();
   
```

@> uint256 amount = \_undeploy(withdrawAmount);
…
```

Therefore, in the `undeploy` process, `deltaCollateralAmount` is in 18-decimal format. It is directly packed into `data` and passed to `_repayAndWithdraw` during the callback.

As a result, the `_withdraw` functions in `StrategyLeverageAAVEv3` and `StrategyLeverageMorphoBlue` should convert the input `amount` from 18-decimal format to the token's actual decimal format. Otherwise, the wrong amount will be withdrawn.

3. In the `_undeploy` process, `deltaDebt` and fees should be converted from 18-decimal format to the `debtToken`'s actual decimal format.

4. The `_convertToCollateral` and `_convertToDebt` functions expect the `amount` parameter to be in 18-decimal format, as required for calculations by `_toDebt` and `_toCollateral` using the oracle. However, before proceeding with the swap, the amount needs to be converted to the respective token's actual decimal format.
Additionally, `_convertToCollateral` receives the token's original decimal `amount` during the deploy process, leading to incorrect calculations by the oracle.

```solidity
    /**
     * @dev Internal function to convert the specified amount from Debt Token to the underlying collateral asset cbETH, wstETH, rETH.
     *
     * This function is virtual and intended to be overridden in derived contracts for customized implementation.
     *
     * @param amount The amount to convert from debtToken.
     * @return uint256 The converted amount in the underlying collateral.
     */
    function _convertToCollateral(uint256 amount) internal virtual returns (uint256) {
        uint256 amountOutMinimum = 0;

        if (getMaxSlippage() > 0) {
            uint256 wsthETHAmount = _toCollateral(
                IOracle.PriceOptions({maxAge: getPriceMaxAge(), maxConf: getPriceMaxConf()}),
                amount,
                false
            );
            amountOutMinimum = (wsthETHAmount * (PERCENTAGE_PRECISION - getMaxSlippage())) / PERCENTAGE_PRECISION;
        }
        // 1. Swap Debt Token -> Collateral Token
        (, uint256 amountOut) = swap(
            ISwapHandler.SwapParams(
                _debtToken, // Asset In
                _collateralToken, // Asset Out
                ISwapHandler.SwapType.EXACT_INPUT, // Swap Mode
                amount, // Amount In
                amountOutMinimum, // Amount Out
                bytes("") // User Payload
            )
        );
        return amountOut;
    }

    /**
     * @dev Internal function to convert the specified amount to Debt Token from the underlying collateral.
     *
     * This function is virtual and intended to be overridden in derived contracts for customized implementation.
     *
     * @param amount The amount to convert to Debt Token.
     * @return uint256 The converted amount in Debt Token.
     */
    function _convertToDebt(uint256 amount) internal virtual returns (uint256) {
        uint256 amountOutMinimum = 0;
        if (getMaxSlippage() > 0) {
            uint256 ethAmount = _toDebt(
                IOracle.PriceOptions({maxAge: getPriceMaxAge(), maxConf: getPriceMaxConf()}),
                amount,
                false
            );
            amountOutMinimum = (ethAmount * (PERCENTAGE_PRECISION - getMaxSlippage())) / PERCENTAGE_PRECISION;
        }
        // 1.Swap Colalteral -> Debt Token
        (, uint256 amountOut) = swap(
            ISwapHandler.SwapParams(
                _collateralToken, // Asset In
                _debtToken, // Asset Out
                ISwapHandler.SwapType.EXACT_INPUT, // Swap Mode
                amount, // Amount In
                amountOutMinimum, // Amount Out
                bytes("") // User Payload
            )
        );
        return amountOut;
    }
```

5. The `_convertToCollateral` and `_convertToDebt` functions default to returning the `amount` in the token’s actual decimal format. However, certain parts of the code assume they return the amount in 18-decimal format, leading to potential miscalculations.
6. The `_adjustDebt` function should convert the flash loan amount from 18-decimal format to the token’s original decimal format.
7. The `_payDebt` function will receive an amount in 18-decimal format, but when performing the swap, the amount is not converted to the token’s actual decimal format. This can lead to incorrect calculations during the swap process.

### Recommended mitigation steps

It is recommended to align the vault’s decimals with the underlying token’s decimals instead of using 18 decimals. This alignment can significantly reduce the complexity of decimal conversions throughout the system.

**chefkenji (BakerFi) confirmed**

**[BakerFi mitigated](https://github.com/code-423n4/2025-01-bakerfi-mitigation?tab=readme-ov-file# findings-being-mitigated):**

> [PR-24](https://github.com/baker-fi/bakerfi-contracts/pull/24)

**Status:** Mitigation confirmed. Full details in reports from [0xlemon](https://code4rena.com/evaluate/2025-01-bakerfi-mitigation-review/findings/S-28) and [shaflow2](https://code4rena.com/evaluate/2025-01-bakerfi-mitigation-review/findings/S-16).

---

---
### Example 2

**Auto Label:** **Incorrect decimal handling leading to precision errors, inaccurate asset calculations, and potential financial losses due to flawed assumptions about token decimal precision.**  

**Original Text Preview:**

## Context

Node.sol#L736-L737

## Description

In function `_calculateSharesAfterSwingPricing()`, there is an evaluation that might be ambiguous.

## Recommendation

To be sure and for readability, consider changing the code to:

```javascript
if (totalAssets() == 0 && totalSupply() == 0 || !swingPricingEnabled) {
```

to 

```javascript
if ((totalAssets() == 0 && totalSupply() == 0) || !swingPricingEnabled) {
```

## NashPoint

Fixed in PR 192:
- Due to the async nature of 7540, there is no clear way to atomically withdraw from a 7540 component and fulfill a redemption.
- A future version of the 7540 router may include this if it is a requested feature.
- Intended operation is for the rebalancer to liquidate a 7540 component into the reserve in order to fulfill requests.

## Cantina Managed

Fix verified.

---
### Example 3

**Auto Label:** Inconsistent decimal handling across contracts leads to arithmetic errors, precision loss, and miscalculated token amounts, risking incorrect balances, dust accumulation, and severe financial misstatements.  

**Original Text Preview:**

##### Description

The `wrap` and `unwrap` functions assume a 1:1 correspondence between `originToken` and the `WrappedToken` amounts, but there is no check to ensure that the decimals of the `originToken` and the `WrappedToken` match. This can lead to unintended and incorrect token amounts being minted or burned, potentially causing financial discrepancies.

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:M/A:N/D:N/Y:N/R:N/S:C (6.3)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:M/A:N/D:N/Y:N/R:N/S:C)

##### Recommendation

1. Perform a check in the `initWToken` function to ensure the decimals of `originToken` match the `DECIMALS` value set during initialization.
2. Extract the decimals directly from the `originToken` using its `decimals()` function and not rely on external parameters.

##### Remediation

**RISK ACCEPTED**: This problem does exist. However, when setting the token, the administrator will pay attention to the parameters.

---
