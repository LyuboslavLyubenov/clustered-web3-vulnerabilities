# Cluster -1211

**Rank:** #288  
**Count:** 26  

## Label
Inaccurate off-chain assumptions and flawed data validation lead to incorrect on-chain calculations, resulting in misleading liquidity assessments, improper token valuations, and potential undercollateralization due to dynamic price ignorance.

## Cluster Information
- **Total Findings:** 26

## Examples

### Example 1

**Auto Label:** Precision errors in collateral calculations lead to incorrect asset valuations, undermining stability, risk assessment, and liquidation mechanisms across protocols.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Low

## Description

The `_redeemCollateralFromTrove()` function, used internally by `redeemCollateral()`, assumes all collateral tokens have 18 decimals, matching the debt token. However, when handling tokens with different precisions, incorrect calculations occur due to this formula:

```solidity
        singleRedemption.collateralLot = (singleRedemption.debtLot * DECIMAL_PRECISION) / _price;
```

Since `singleRedemption.debtLot`, `_price`, and `DECIMAL_PRECISION` are all 18-decimal values, this results in an invalid collateralLot for tokens with different decimals. This leads to:

- Underflows due to precision mismatches.
- Incorrect collateral balances, causing miscalculations in accounting.

Example issue in `newColl` calculation:

```solidity
        uint256 newColl = (t.coll) - singleRedemption.collateralLot;
```

Additionally, `_updateBaseRateFromRedemption()` is affected, as `_CollateralDrawn` should match the collateral’s precision, while \_price and `_totalDebtSupply` remain in 1e18 precision.

## Recommendations

Ensure all calculations are normalized to a consistent precision(i.e. 18 decimals).

---
### Example 2

**Auto Label:** Inaccurate state calculations due to flawed balance aggregation and missing external price or interest validation, leading to misleading liquidity estimates and unauthorized pool manipulation.  

**Original Text Preview:**

## Severity: Low Risk

## Context
AaveV3Farm.sol#L69-L72

## Description
In the `AaveV3.ValidationLogic` contract, the maximum deposit calculation considers the `accruedToTreasury` variable when checking against `supplyCap`:

```solidity
require(
    supplyCap == 0 ||
    ((IAToken(reserveCache.aTokenAddress).scaledTotalSupply() +
    uint256(reserve.accruedToTreasury)).rayMul(reserveCache.nextLiquidityIndex) + amount) >=
    supplyCap * (10 ** reserveCache.reserveConfiguration.getDecimals()),
    Errors.SUPPLY_CAP_EXCEEDED
);
```

However, the `AaveV3Farm` contract calculates the maximum deposit using only `aToken.totalSupply()`, while it should also consider the amount accrued to treasury. By returning an incorrect maximum deposit, it could revert when trying.

## Recommendation
Consider changing the code to the following:

```solidity
uint256 currentUnderlyingProtocolSupply = ERC20(aToken).totalSupply();
return supplyCapInAssetTokenDecimals - currentUnderlyingProtocolSupply;
```

Change to:

```solidity
(, uint256 _accruedToTreasuryScaled, uint256 _totalAToken, , , , , , , , ,) =
IAaveDataProvider(dataProvider).getReserveData(address(asset));

// Supply cap already reached
if (_totalAToken + _accruedToTreasuryScaled >= _supplyCap) {
    return 0;
}
return supplyCapInAssetTokenDecimals - (_totalAToken + _accruedToTreasuryScaled);
```

## Additional Notes
- **infiniFi**: Fixed in `0b20a51`.
- **Spearbit**: Fix verified.

---
### Example 3

**Auto Label:** Inaccurate state calculations due to flawed balance aggregation and missing external price or interest validation, leading to misleading liquidity estimates and unauthorized pool manipulation.  

**Original Text Preview:**

**Impact**

`addLiquidity` looks as follows

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/SwapOperations.sol#L228-L246

```solidity
  function addLiquidity(
    address tokenA,
    address tokenB,
    uint amountADesired,
    uint amountBDesired,
    uint amountAMin,
    uint amountBMin,
    PriceUpdateAndMintMeta memory _priceAndMintMeta,
    uint deadline
  ) public payable virtual override ensure(deadline) returns (uint amountA, uint amountB, uint liquidity) {
    ProvidingVars memory vars;
    vars.pair = getPair[tokenA][tokenB]; /// @audit QA not sorting tokens means this can revert | No fix is ok but it's a gotcha
    if (vars.pair == address(0)) revert PairDoesNotExist();
    /// @audit not checking that min < desired - S
    {
      (vars.reserveA, vars.reserveB) = getReserves(tokenA, tokenB);
      if (vars.reserveA == 0 && vars.reserveB == 0) {
        (amountA, amountB) = (amountADesired, amountBDesired); /// @audit First LP Can attack by massively imbalancing by performing a single sided swap or similar
      } else {
```

When no liquidity was previously added, the ratio for the LP will be taken at face value

This could be used by the first depositor to purposefully imbalance the pool, as a means to take on debts that either allow them to be closer to delta neutral, or as a means to grief other users

**Mitigation**

Consider using a ratio that is taken from the oracles

---
