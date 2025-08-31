# Cluster -1457

**Rank:** #180  
**Count:** 68  

## Label
Inaccurate mathematical calculations in pricing and rate computations due to flawed arithmetic logic or improper precision handling, leading to erroneous asset valuations, flawed trade execution, and potential financial loss.

## Cluster Information
- **Total Findings:** 68

## Examples

### Example 1

**Auto Label:** Inaccurate reserve or fee calculations due to flawed mathematical logic or improper accounting, enabling price manipulation, liquidity distortion, and unauthorized fund manipulation.  

**Original Text Preview:**

**Impact**

At a broadstroke The formula to compute the post-swap reserves used by `getAmountsOut` and `getAmountsIn` is as follows:

```
uint(reserveA) * reserveB) / (reserveB + amtB),
```

However, the math is not accounting for fees that will be taken on each swap, meaning that the post-swap reserves will not match this amount


**Mitigation**

You should technically also account for the updated price post taking fees, as that will be the price that the next person will pay

I don't have sufficient time to fully explore this issue and I highly recommend you simulate swaps to determine if this can cause significant losses

---
### Example 2

**Auto Label:** **Incorrect price or rate calculation due to flawed mathematical logic or stale data, enabling manipulation, arbitrage, or value leakage.**  

**Original Text Preview:**

## Summary

The `StabilityPool::getExchangeRate()` function is hardcoded to return a fixed value of `1e18`, which assumes a static 1:1 exchange rate between `rToken and deToken`. This implementation does not account for real-time changes in token supply and demand. As a result, users may receive incorrect amounts when depositing or redeeming tokens, leading to potential financial imbalances and arbitrage exploits.

## Vulnerability Details

```Solidity
function getExchangeRate() public view returns (uint256) {
    // uint256 totalDeCRVUSD = deToken.totalSupply();
    // uint256 totalRcrvUSD = rToken.balanceOf(address(this));
    // if (totalDeCRVUSD == 0 || totalRcrvUSD == 0) return 10**18;

    // uint256 scalingFactor = 10**(18 + deTokenDecimals - rTokenDecimals);
    // return (totalRcrvUSD * scalingFactor) / totalDeCRVUSD;
    return 1e18;
}
```

### **Issue:**

1. **Static Exchange Rate:**

   * The function always returns `1e18`, meaning the exchange rate between `rToken` and `deToken` is assumed to be fixed at 1:1.
   * The commented-out code suggests an original intention to make the rate dynamic based on token supply but was removed.

2. **Incorrect Deposit and Redemption Calculations:**

   * The deposit function (`deposit()`) relies on `calculateDeCRVUSDAmount()`, which uses `getExchangeRate()`.
   * The redemption function (`calculateRcrvUSDAmount()`) also uses `getExchangeRate()`.
   * Since `getExchangeRate()` is hardcoded, these calculations may not reflect real market conditions.

3. **Arbitrage Risks:**

   * If the real market value of `rToken` fluctuates, users can **deposit undervalued tokens and redeem overvalued ones**, extracting unearned profits.
   * This could result in **protocol insolvency** if redemptions exceed available reserves.

4. **Liquidity Imbalance:**

   * The total supply of `deToken` may become misaligned with the actual available `rToken` balance.
   * If too many deposits occur before an update, redemptions may fail due to insufficient reserves.



## Impact

<https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/pools/StabilityPool/StabilityPool.sol#L211-L212>

**Real-World Impact** A similar issue occurred in **DeFi lending protocols**, where mispriced exchange rates led to excessive borrowing and redemption exploits. A well-known case was the **Iron Finance Bank Run**, where an artificially pegged stablecoin lost parity, causing mass redemptions and liquidity depletion.

## Tools Used

## Recommendations

Replace the hardcoded `1e18` with a formula that correctly derives the rate based on `deToken.totalSupply()` and `rToken.balanceOf(address(this))`.

---
### Example 3

**Auto Label:** **Incorrect price or rate calculation due to flawed mathematical logic or stale data, enabling manipulation, arbitrage, or value leakage.**  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

During the deposit, code calls `calculateTokenRatio()` to calculate the `ratio0` and `ratio1`. If the pool's prices reaches the upper or lower price then `ratio0` or `ratio1` would be 0:

```solidity
        } else if (price <= priceLower) {
            ratio0 = 0;
            ratio1 = decimals1;
        } else if (price >= priceUpper) {
            ratio0 = decimals0;
            ratio1 = 0;
        }
```

Later, code uses the `ratio0` and `ratio1` to calculate the max deposit amount by calling `calculateMaxDeposit()`. The issue is that if `ratio0` or `ratio1` were 0 then function `calculateMaxDeposit()` can revert because of division by zero:

```solidity
    function calculateMaxDeposit(bool isToken0, uint256 amount0, uint256 amount1, uint256 ratio0, uint256 ratio1)
        internal view returns(uint256 deposit0, uint256 deposit1) {
        if(isToken0) {
            deposit0 = amount0;
            deposit1 = ratio1 * amount0 / ratio0;
            if(deposit1 > amount1) {
                deposit0 = ratio0 * amount1 / ratio1;
                deposit1 = ratio1 * deposit0 / ratio0;
            }
        } else {
            deposit1 = amount1;
            deposit0 = ratio0 * amount1 / ratio1;
            if(deposit0 > amount0) {
                deposit1 = ratio1 * amount0 / ratio0;
                deposit0 = ratio0 * deposit1 / ratio1;
            }
        }
    }
```

As result when price is equal to min or max tick the deposit would revert always.

## Recommendations

Handle the case when ratio0 or raio1 is 0 in the `calculateMaxDeposit()`

---
