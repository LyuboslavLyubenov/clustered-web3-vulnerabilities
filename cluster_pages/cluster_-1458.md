# Cluster -1458

**Rank:** #446  
**Count:** 8  

## Label
Precision loss in fixed-point arithmetic due to inconsistent decimal scaling and floating-point conversions, leading to incorrect comparisons, inaccurate calculations, and flawed decision logic.

## Cluster Information
- **Total Findings:** 8

## Examples

### Example 1

**Auto Label:** Precision loss in fixed-point arithmetic due to inconsistent decimal scaling and floating-point conversions, leading to incorrect comparisons, inaccurate calculations, and flawed decision logic.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-12-plaza-finance-judging/issues/561 

## Found by 
0x52, 0xadrii, 0xc0ffEE, Hueber, Ryonen, X0sauce, ZoA, bretzel, farman1094, future, fuzzysquirrel, shui, stuart\_the\_minion, tinnohofficial

### Summary

A decimal precision mismatch between `marketRate ` (18 decimal precision) and `redeemRate ` (6 decimal precision) in `Pool.sol` will cause the market rate to never be used.

### Root Cause

In [`Pool.sol#L512-L516`](https://github.com/sherlock-audit/2024-12-plaza-finance/blob/main/plaza-evm/src/Pool.sol#L512-L516), the `redeemRate` is calculated and implicitly uses a precision of 6 decimal precision : 
- [Pool#512](https://github.com/sherlock-audit/2024-12-plaza-finance/blob/main/plaza-evm/src/Pool.sol#L512)
- [Pool#516](https://github.com/sherlock-audit/2024-12-plaza-finance/blob/main/plaza-evm/src/Pool.sol#L514)
- [Pool#516](https://github.com/sherlock-audit/2024-12-plaza-finance/blob/main/plaza-evm/src/Pool.sol#L516) : The constant `BOND_TARGET_PRICE = 100` multiplied by `PRECISION = 1e6` = 100e6.

However, the `marketRate` will be [normalized](https://github.com/sherlock-audit/2024-12-plaza-finance/blob/main/plaza-evm/src/Pool.sol#L447) to 18dp : 
       The BPT token itself has 18 decimals ([`BalancerPoolToken.sol`](https://github.com/balancer/balancer-v2-monorepo/blob/master/pkg/pool-utils/contracts/BalancerPoolToken.sol)) so `totalSupply()` is 18dp.
     When calculating the price of a BPT it will formalize each price of the asset of the BPT pool to 18dp :  "balancer math works with 18 dec" [BalancerOracleAdapter.sol#L109](https://github.com/sherlock-audit/2024-12-plaza-finance/blob/main/plaza-evm/src/BalancerOracleAdapter.sol#L109). 
    It implies that the `decimals` of  [BalancerOracleAdapter](https://github.com/sherlock-audit/2024-12-plaza-finance/blob/main/plaza-evm/src/BalancerOracleAdapter.sol#L51) is set to 18.
Then the final value will have a precision of 18dp.

The comparison `marketRate < redeemRate` will always be false due to this difference in decimal precision.

### Internal Pre-conditions

1. A Chainlink price feed for the bond token must exist and be registered in `OracleFeeds`.
2. The `marketRate` from the Balancer oracle is lower than the calculated `redeemRate` when both are expressed with the same decimal precision.
3. `getOracleDecimals(reserveToken, USD)` returns 18

### External Pre-conditions

N/A

### Attack Path

1. A user initiates a redeem transaction.
2. The `simulateRedeem` and `getRedeemAmount` functions are called.
3. The condition `marketRate < redeemRate` evaluates to false due to the decimal mismatch.
4. The `redeemRate`, which might be higher than the actual market rate, is used to calculate the amount of reserve tokens the user receives.

### Impact

The intended functionality of considering the market rate for redemptions is completely bypassed.
Users redeeming tokens might receive more reserve tokens than expected if the true market rate (with correct decimals) is lower than the calculated `redeemRate`.

### PoC

N/A

### Mitigation

Change the normalization in `simulateRedeem` to use the  `bondToken.SHARES_DECIMALS()` instead of `oracleDecimals`.

```solidity
uint256 marketRate;
address feed = OracleFeeds(oracleFeeds).priceFeeds(address(bondToken), USD);
uint8 sharesDecimals = bondToken.SHARES_DECIMALS(); // Get the decimals of the shares

if (feed != address(0)) {
    marketRate = getOraclePrice(address(bondToken), USD).normalizeAmount(
        getOracleDecimals(address(bondToken), USD), 
        sharesDecimals // Normalize to sharesDecimals
    );
}
```

Modify the normalization of `marketRate` in `Pool.sol`'s `simulateRedeem` function to use the same decimal precision as `redeemRate` (6 decimals).  Specifically, change the normalization to use `bondToken.SHARES_DECIMALS()` instead of `oracleDecimals`:

```diff
 if (feed != address(0)) {
+ uint8 sharesDecimals = bondToken.SHARES_DECIMALS(); // Use sharesDecimals for consistent precision
  marketRate = getOraclePrice(address(bondToken), USD)
        .normalizeAmount(
          getOracleDecimals(address(bondToken), USD),
-          oracleDecimals // this is the decimals of the reserve token chainlink feed
+         sharesDecimals
        );


 }
 return getRedeemAmount(tokenType, depositAmount, bondSupply, levSupply, poolReserves, getOraclePrice(reserveToken, USD), oracleDecimals, marketRate)
         .normalizeAmount(COMMON_DECIMALS, IERC20(reserveToken).safeDecimals());

```

## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/Convexity-Research/plaza-evm/pull/156

---
### Example 2

**Auto Label:** Precision errors in fixed-point arithmetic cause incorrect calculations, leading to yield loss, free token issuance, or price distortion due to improper rounding or order of operations.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-10-mento-update-judging/issues/21 

## Found by 
0x73696d616f
### Summary

[GoodDollarExchangeProvider::mintFromExpansion()](https://github.com/sherlock-audit/2024-10-mento-update/blob/main/mento-core/contracts/goodDollar/GoodDollarExchangeProvider.sol#L147) mints supply tokens while keeping the current price constant. To achieve this, a certain formula is used, but in the process it scales the `reserveRatioScalar * exchange.reserveRatio` to `1e8` precision (the precision of `exchange.reserveRatio`) down from `1e18`. 

However, the calculation of the new amount of tokens to mint is based on the full ratio with 1e18, which will mint more tokens than it should and change the price, breaking the readme.

Note1: there is also a slight price change in [GoodDollarExchangeProvider::mintFromInterest()](https://github.com/sherlock-audit/2024-10-mento-update/blob/main/mento-core/contracts/goodDollar/GoodDollarExchangeProvider.sol#L179-L181) due to using `mul` and then `div`, as `mul` divides by `1e18` unnecessarily in this case.

Note2: [GoodDollarExchangeProvider::updateRatioForReward()](https://github.com/sherlock-audit/2024-10-mento-update/blob/main/mento-core/contracts/goodDollar/GoodDollarExchangeProvider.sol#L205) also has precision loss as it calculates the ratio using the formula and then scales it down, changing the price.

### Root Cause

In `GoodDollarExchangeProvider:147`, `newRatio` is calculated with full `1e18` precision and used to calculate the amount of tokens to mint, but `exchanges[exchangeId].reserveRatio` is stored with the downscaled value, `newRatio / 1e10`, causing an error and price change. 

This happens because the price is `reserve / (supply * reserveRatio)`. As `supply` is increased by a calculation that uses the full precision `newRatio`, but `reserveRatio` is stored with less precision (`1e8`), the price will change due to this call.

### Internal pre-conditions

None.

### External pre-conditions

None.

### Attack Path

1. `GoodDollarExchangeProvider::mintFromExpansion()` is called and a rounding error happens in the calculation of `newRatio`.

### Impact

The current price is modified due to the expansion which goes against the readme:
> What properties/invariants do you want to hold even if breaking them has a low/unknown impact?

> Bancor formula invariant. Price = Reserve / Supply * reserveRatio

### PoC

Add the following test to `GoodDollarExchangeProvider.t.sol`:
```solidity
function test_POC_mintFromExpansion_priceChangeFix() public {
  uint256 priceBefore = exchangeProvider.currentPrice(exchangeId);
  vm.prank(expansionControllerAddress);
  exchangeProvider.mintFromExpansion(exchangeId, reserveRatioScalar);
  uint256 priceAfter = exchangeProvider.currentPrice(exchangeId);
  assertEq(priceBefore, priceAfter, "Price should remain exactly equal");
}
```
If the code is used as is, it fails. but if it is fixed by dividing and multiplying by `1e10`, eliminating the rounding error, the price matches exactly (exact fix show below).

### Mitigation

Divide and multiply `newRatio` by `1e10` to eliminate the rounding error, keeping the price unchanged.
```solidity
function mintFromExpansion(
  bytes32 exchangeId,
  uint256 reserveRatioScalar
) external onlyExpansionController whenNotPaused returns (uint256 amountToMint) {
  require(reserveRatioScalar > 0, "Reserve ratio scalar must be greater than 0");
  PoolExchange memory exchange = getPoolExchange(exchangeId);

  UD60x18 scaledRatio = wrap(uint256(exchange.reserveRatio) * 1e10);
  UD60x18 newRatio = wrap(unwrap(scaledRatio.mul(wrap(reserveRatioScalar))) / 1e10 * 1e10);
  ...
}
```

---
### Example 3

**Auto Label:** Precision loss in fixed-point arithmetic due to inconsistent decimal scaling and floating-point conversions, leading to incorrect comparisons, inaccurate calculations, and flawed decision logic.  

**Original Text Preview:**

The value of `tokenRatio` is [stored in the `GasPriceOracle.sol` contract](https://github.com/mantlenetworkio/mantle-v2/blob/e29d360904db5e5ec81888885f7b7250f8255895/packages/contracts-bedrock/contracts/L2/GasPriceOracle.sol#L29) on L2 as a `uint256`. This `tokenRatio` is intended to take on the quotient value of ETH price divided by MNT price, and it is used to calculate the gas on L2. Currently, when this value is updated, all decimals are truncated. Given the current market price of ETH and MNT, the truncation would only result in a slight error in the gas calculation. However, if the value of MNT grows relative to the price of ETH, the error will continue to grow.


Therefore, consider scaling up the `tokenRatio` value and, upon using this value in op-geth, scale it back down to its proper scale at that point in order to improve precision.


***Update:** Acknowledged, will resolve.*

---
