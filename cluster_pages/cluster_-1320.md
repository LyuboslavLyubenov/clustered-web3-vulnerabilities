# Cluster -1320

**Rank:** #371  
**Count:** 15  

## Label
Flawed mathematical modeling in swap calculations leads to incorrect pricing, non-monotonic outputs, and exploitable rounding errors, enabling profit extraction or state manipulation.

## Cluster Information
- **Total Findings:** 15

## Examples

### Example 1

**Auto Label:** Inconsistent mathematical modeling in liquidity density calculations leads to precision errors and erroneous swap computations due to flawed rounding and improper function inversion.  

**Original Text Preview:**

**Description:** While there has been some effort made toward systematic rounding, there remain a number of instances where the handling and/or direction of rounding is inconsistent. The first example is within `LibCarpetedGeometricDistribution::liquidityDensityX96` which rounds the returned carpet liquidity up:

```solidity
return carpetLiquidity.divUp(uint24(numRoundedTicksCarpeted));
```

Meanwhile, `LibCarpetedDoubleGeometricDistribution::liquidityDensityX96` rounds down:

```solidity
return carpetLiquidity / uint24(numRoundedTicksCarpeted);
```

Additionally, a configuration was identified that allowed the addition of density contributions from all ricks to exceed `Q96`. Since it is only the current rick liquidity that is used in computations, this does not appear to be a problem as it will simply account a negligibly small amount of additional liquidity density.

**Proof of Concept:** The following test can be run from within `GeometricDistribution.t.sol`:

```solidity
function test_liquidityDensity_sumUpToOneGeometricDistribution(int24 tickSpacing, int24 minTick, int24 length, uint256 alpha)
    external
    virtual
{
    alpha = bound(alpha, MIN_ALPHA, MAX_ALPHA);
    vm.assume(alpha != 1e8); // 1e8 is a special case that causes overflow
    tickSpacing = int24(bound(tickSpacing, MIN_TICK_SPACING, MAX_TICK_SPACING));
    (int24 minUsableTick, int24 maxUsableTick) =
        (TickMath.minUsableTick(tickSpacing), TickMath.maxUsableTick(tickSpacing));
    minTick = roundTickSingle(int24(bound(minTick, minUsableTick, maxUsableTick - 2 * tickSpacing)), tickSpacing);
    length = int24(bound(length, 1, (maxUsableTick - minTick) / tickSpacing - 1));

    console2.log("alpha", alpha);
    console2.log("tickSpacing", tickSpacing);
    console2.log("minTick", minTick);
    console2.log("length", length);

    PoolKey memory key;
    key.tickSpacing = tickSpacing;
    bytes32 ldfParams = bytes32(abi.encodePacked(ShiftMode.STATIC, minTick, int16(length), uint32(alpha)));
    vm.assume(ldf.isValidParams(key, 0, ldfParams));


    uint256 cumulativeLiquidityDensity;
    (int24 minTick, int24 maxTick) =
        (TickMath.minUsableTick(tickSpacing), TickMath.maxUsableTick(tickSpacing) - tickSpacing);
    key.tickSpacing = tickSpacing;
    int24 spotPriceTick = 0;
    for (int24 tick = minTick; tick <= maxTick; tick += tickSpacing) {
        (uint256 liquidityDensityX96,,,,) = ldf.query(key, tick, 0, spotPriceTick, ldfParams, LDF_STATE);
        cumulativeLiquidityDensity += liquidityDensityX96;
    }

    console2.log("Cumulative liquidity density", cumulativeLiquidityDensity);
    console2.log("One", FixedPoint96.Q96);
    if(cumulativeLiquidityDensity > FixedPoint96.Q96) console2.log("Extra cumulative liquidity density", cumulativeLiquidityDensity - FixedPoint96.Q96);
    assertTrue(cumulativeLiquidityDensity <= FixedPoint96.Q96);
}
```

**Recommended Mitigation:** Further clarify opaque rounding decisions with inline comments and standardize any inconsistencies such as those noted above.

**Bacon Labs:** Fixed the LibCarpetedDoubleGeometricDistribution rounding issue in [PR \#127](https://github.com/timeless-fi/bunni-v2/pull/127). Agreed that the total density slightly exceeding `Q96` is fine.

**Cyfrin:** Verified, carpet liquidity is now rounded up in `LibCarpetedDoubleGeometricDistribution::liquidityDensityX96`.

---
### Example 2

**Auto Label:** Inconsistent mathematical modeling in liquidity density calculations leads to precision errors and erroneous swap computations due to flawed rounding and improper function inversion.  

**Original Text Preview:**

**Description:** `QueryLDF::queryLDF` takes an argument `idleBalance` that is not currently but should be included in the function NatSpec.

```solidity
/// @notice Queries the liquidity density function for the given pool and tick
/// @param key The pool key
/// @param sqrtPriceX96 The current sqrt price of the pool
/// @param tick The current tick of the pool
/// @param arithmeticMeanTick The TWAP oracle value
/// @param ldf The liquidity density function
/// @param ldfParams The parameters for the liquidity density function
/// @param ldfState The current state of the liquidity density function
/// @param balance0 The balance of token0 in the pool
/// @param balance1 The balance of token1 in the pool
/// @return totalLiquidity The total liquidity in the pool
/// @return totalDensity0X96 The total density of token0 in the pool, scaled by Q96
/// @return totalDensity1X96 The total density of token1 in the pool, scaled by Q96
/// @return liquidityDensityOfRoundedTickX96 The liquidity density of the rounded tick, scaled by Q96
/// @return activeBalance0 The active balance of token0 in the pool, which is the amount used by swap liquidity
/// @return activeBalance1 The active balance of token1 in the pool, which is the amount used by swap liquidity
/// @return newLdfState The new state of the liquidity density function
/// @return shouldSurge Whether the pool should surge
function queryLDF(
    PoolKey memory key,
    uint160 sqrtPriceX96,
    int24 tick,
    int24 arithmeticMeanTick,
    ILiquidityDensityFunction ldf,
    bytes32 ldfParams,
    bytes32 ldfState,
    uint256 balance0,
    uint256 balance1,
    IdleBalance idleBalance
)
```

**Bacon Labs:** Fixed in [PR \#110](https://github.com/timeless-fi/bunni-v2/pull/110).

**Cyfrin:** Verified, `idleBalance` has been added to the `queryLDF()` NatSpec.

---
### Example 3

**Auto Label:** Flawed mathematical modeling in swap calculations leads to incorrect pricing, non-monotonic outputs, and exploitable rounding errors, enabling profit extraction or state manipulation.  

**Original Text Preview:**

`reverse_simulate_swap_operations` function incorrectly uses `query_simulation` instead of `query_reverse_simulation` when calculating multi-hop trades in reverse. As a result, users will get incorrect price calculations when they attempt to determine how many input tokens they need for a desired output amount.

In summary:

1. Users receive incorrect price quotes for trades.
2. The error compounds in multi-hop trades, causing issues.

### Proof of Concept

[/contracts/pool-manager/src/queries.rs# L275](https://github.com/code-423n4/2024-11-mantra-dex/blob/26714ea59dab7ecfafca9db1138d60adcf513588/contracts/pool-manager/src/queries.rs# L275)

The bug is in the `reverse_simulate_swap_operations` function:
```

pub fn reverse_simulate_swap_operations(
    deps: Deps,
    ask_amount: Uint128,
    operations: Vec<SwapOperation>,
) -> Result<SimulateSwapOperationsResponse, ContractError> {
    let operations_len = operations.len();
    if operations_len == 0 {
        return Err(ContractError::NoSwapOperationsProvided);
    }

    let mut amount = ask_amount;

    for operation in operations.into_iter().rev() {
        match operation {
            SwapOperation::MantraSwap {
                token_in_denom,
                token_out_denom,
                pool_identifier,
            } => {
>>>             let res = query_simulation(
                    deps,
                    coin(amount.u128(), token_out_denom),
                    token_in_denom,
                    pool_identifier,
                )?;
                amount = res.return_amount;
            }
        }
    }

    Ok(SimulateSwapOperationsResponse { amount })
}
```

[According to the docs](https://docs.mantrachain.io/mantra-smart-contracts/mantra_dex/pool-manager# reversesimulation) and the example given in it:

We have a `QUERY`:
```

{
  "reverse_simulation": {
    "ask_asset": {
      "denom": "uusdc",
      "amount": "990000"
    },
    "offer_asset_denom": "uom",
    "pool_identifier": "om-usdc-1"
  }
}
```

And a `RESPONSE` (`ReverseSimulationResponse`):
```

{
  "offer_amount": "1000000",
  "spread_amount": "5000",
  "swap_fee_amount": "3000",
  "protocol_fee_amount": "2000",
  "burn_fee_amount": "0",
  "extra_fees_amount": "0"
}
```

The doc also clearly states that `offer_amount` in the response is “The amount of the offer asset needed to get the ask amount.”

Now, let’s compare again with the implementation:
```

let res = query_simulation(  <--- This is WRONG! Its using forward simulation
    deps,
    coin(amount.u128(), token_out_denom),
    token_in_denom,
    pool_identifier,
)?;
```

The bug is clear because it’s using `query_simulation` instead of `query_reverse_simulation`.

Also consider a multi-hop trade `A -> B -> C` where a user wants 100 C tokens:

1. The fn calculates how many `B` tokens you get for `X C` tokens (forward).
2. Then calculates how many `A` tokens you get for `Y B` tokens (forward).
3. This is incorrect for reverse price discovery.

It shoould actually be:

1. Calculate how many `B` tokens you need to get `100 C` tokens (reverse).
2. Then calculate how many `A` tokens you need to get the required `B` tokens (reverse).

The contract already has a correct `query_reverse_simulation` function that handles proper price calculation for reverse operations, but it’s not being used.

### Recommended mitigation steps
```

pub fn reverse_simulate_swap_operations(
    deps: Deps,
    ask_amount: Uint128,
    operations: Vec<SwapOperation>,
) -> Result<SimulateSwapOperationsResponse, ContractError> {
    let operations_len = operations.len();
    if operations_len == 0 {
        return Err(ContractError::NoSwapOperationsProvided);
    }

    let mut amount = ask_amount;

    for operation in operations.into_iter().rev() {
        match operation {
            SwapOperation::MantraSwap {
                token_in_denom,
                token_out_denom,
                pool_identifier,
            } => {
-                let res = query_simulation(
+                let res = query_reverse_simulation(
                    deps,
                    coin(amount.u128(), token_out_denom),
                    token_in_denom,

                    pool_identifier,
                )?;
-                amount = res.return_amount;
+                amount = res.offer_amount;
            }
        }
    }

    Ok(SimulateSwapOperationsResponse { amount })
}
```

**[jvr0x (MANTRA) disputed and commented](https://code4rena.com/audits/2024-11-mantra-dex/submissions/F-68?commentParent=rrfx3Hnvjwb):**

> This is something that was fixed in a subsequent commit after the v1.0.0 tag was created. The current (live) reverse query code is the following:
>
> 
```

> pub fn reverse_simulate_swap_operations(
>     deps: Deps,
>     ask_amount: Uint128,
>     operations: Vec<SwapOperation>,
> ) -> Result<ReverseSimulateSwapOperationsResponse, ContractError> {
>     let operations_len = operations.len();
>     if operations_len == 0 {
>         return Err(ContractError::NoSwapOperationsProvided);
>     }
>
>     let mut offer_in_needed = ask_amount;
>     let mut spreads: Vec<Coin> = vec![];
>     let mut swap_fees: Vec<Coin> = vec![];
>     let mut protocol_fees: Vec<Coin> = vec![];
>     let mut burn_fees: Vec<Coin> = vec![];
>     let mut extra_fees: Vec<Coin> = vec![];
>
>     for operation in operations.into_iter().rev() {
>         match operation {
>             SwapOperation::MantraSwap {
>                 token_in_denom,
>                 token_out_denom,
>                 pool_identifier,
>             } => {
>                 let res = query_reverse_simulation(
>                     deps,
>                     coin(offer_in_needed.u128(), token_out_denom.clone()),
>                     token_in_denom,
>                     pool_identifier,
>                 )?;
>
>                 if res.spread_amount > Uint128::zero() {
>                     spreads.push(coin(res.spread_amount.u128(), &token_out_denom));
>                 }
>                 if res.swap_fee_amount > Uint128::zero() {
>                     swap_fees.push(coin(res.swap_fee_amount.u128(), &token_out_denom));
>                 }
>                 if res.protocol_fee_amount > Uint128::zero() {
>                     protocol_fees.push(coin(res.protocol_fee_amount.u128(), &token_out_denom));
>                 }
>                 if res.burn_fee_amount > Uint128::zero() {
>                     burn_fees.push(coin(res.burn_fee_amount.u128(), &token_out_denom));
>                 }
>                 if res.extra_fees_amount > Uint128::zero() {
>                     extra_fees.push(coin(res.extra_fees_amount.u128(), &token_out_denom));
>                 }
>
>                 offer_in_needed = res.offer_amount;
>             }
>         }
>     }
>
>     spreads = aggregate_coins(spreads)?;
>     swap_fees = aggregate_coins(swap_fees)?;
>     protocol_fees = aggregate_coins(protocol_fees)?;
>     burn_fees = aggregate_coins(burn_fees)?;
>     extra_fees = aggregate_coins(extra_fees)?;
>
>     Ok(ReverseSimulateSwapOperationsResponse {
>         offer_amount: offer_in_needed,
>         spreads,
>         swap_fees,
>         protocol_fees,
>         burn_fees,
>         extra_fees,
>     })
> }
> 
```

**[3docSec (judge) commented](https://code4rena.com/audits/2024-11-mantra-dex/submissions/F-68?commentParent=9Bq9c7Tsz2j):**

> For judging, we base on the commit frozen for the audit scope; so while it’s good to see the team found and fixed the issue independently, it wouldn’t be fair to wardens to not reward a valid finding.

---

---
