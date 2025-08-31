# Cluster -1456

**Rank:** #252  
**Count:** 37  

## Label
Common vulnerability type: **Arithmetic errors due to unhandled zero divisions or improper ratio calculations, leading to lost assets, incorrect token distributions, or denied user actions.**

## Cluster Information
- **Total Findings:** 37

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

**Auto Label:** Division-by-zero in ratio calculations due to unvalidated reserve or supply values, leading to function aborts, incorrect computations, or permanent fund locking.  

**Original Text Preview:**

<https://github.com/code-423n4/2025-04-cabal/blob/5b5f92ab4f95e5f9f405bbfa252860472d164705/sources/cabal.move# L996-L998>

<https://github.com/code-423n4/2025-04-cabal/blob/5b5f92ab4f95e5f9f405bbfa252860472d164705/sources/cabal.move# L1051-L1053>

### Finding description and impact

When a user burns the **entire remaining supply** of a Cabal LST ( sxINIT or Cabal LPT) via `initiate_unstake`, the follow‑up processing step always aborts with a divide‑by‑zero and the user can never exit.

1. **User calls `initiate_unstake(stake_type, S)` – S equals the whole supply.**
2. `unstake_xinit` / `unstake_lp` queues `process_*_unstake` with `cosmos::move_execute( … "process_xinit_unstake" | "process_lp_unstake" … )` for next transaction.
3. **After queuing**, `initiate_unstake` burns the LST: `cabal_token::burn(S)` ⇒ live supply becomes **0**.
4. Transaction 1 finishes and state now shows `supply = 0`, `pending[i] = S`.
5. **Later, Transaction 2 executes `process_*_unstake`.**
6. Calls `compound_*_pool_rewards` (does not change LST supply).
7. Reads the current LST supply: `sx_supply = fungible_asset::supply(meta)` ⇒ **0**.
8. Calculates `ratio = bigdecimal::from_ratio_u128(unstake_amount, sx_supply)` which triggers `assert!(denominator != 0)` → **`EDIVISION_BY_ZERO` abort**.

Because the burn happened in a prior committed transaction, every retry of `process_*_unstake` gets the same `supply == 0` state and fails again, so the user’s INIT / LP is permanently locked and it makes a DoS for the final staker of that pool.
```

    // Simplified logic from process_xinit_unstake
    entry fun process_xinit_unstake(account: &signer, staker_addr: address, unstaking_type: u64, unstake_amount: u64) acquires ModuleStore, CabalStore, LockExempt {
        // ... permission checks, reward compounding ...
        let m_store = borrow_global_mut<ModuleStore>(@staking_addr);
        let x_init_amount = m_store.staked_amounts[unstaking_type];

        // --- VULNERABILITY ---
        // 'unstake_amount' is the original amount burned (== total supply in this case).
        // 'sx_init_amount' reads the supply *after* the burn in initiate_unstake, so it's 0.
        let sx_init_amount = option::extract(&mut fungible_asset::supply(m_store.cabal_stake_token_metadata[unstaking_type])); // Returns 0

        // This attempts bigdecimal::from_ratio_u128(S, 0) --> Division by Zero!
        let ratio = bigdecimal::from_ratio_u128(unstake_amount as u128, sx_init_amount);
        // Transaction reverts here.
        // ... rest of function is unreachable ...
    }
```

**Impact:**

If an address burns the last sxINIT / LPT in circulation, every call to `process_*_unstake` reverts with `EDIVISION_BY_ZERO`, so no `UnbondingEntry` is recorded and the underlying INIT / LP can never be claimed. The final staker’s funds are permanently locked and causes a pool‑level denial of service.

### Recommended mitigation steps

In `process_xinit_unstake` and `process_lp_unstake`:
```

let pool_before = m_store.staked_amounts[pool];
let supply      = fungible_asset::supply(meta);

let unbond = if supply == 0 {
    // last holder – give them the entire pool
    pool_before
} else {
    let r = bigdecimal::from_ratio_u128(unstake_amount, supply);
    bigdecimal::mul_by_u64_truncate(r, pool_before)
};
```

* Guard against `supply == 0`.
* If it’s the final unstake, transfer the whole remaining pool; otherwise keep the original ratio logic.

### Proof of Concept
```

// Assume pool index 1 is an LP‑staking pool
let pool_idx: u64 = 1;

// ── step 1: mint exactly 1 Cabal‑LPT to Alice ───────────────────────────
let mint_cap = &ModuleStore.cabal_stake_token_caps[pool_idx].mint_cap;
cabal_token::mint_to(mint_cap, @alice, 1);          // total supply = 1

// ── step 2: Alice initiates unstake of the ENTIRE supply ────────────────
cabal::initiate_unstake(&signer(@alice), pool_idx, 1);
/*
 * inside initiate_unstake:
 *   • cabal_token::burn(1)            → total supply becomes 0
 *   • schedules process_lp_unstake()  (async)
 */

// ── step 3: worker executes queued call ──────────────────────────────────
cabal::process_lp_unstake(&assets_signer, @alice, pool_idx, 1);
/*
 * inside process_lp_unstake:
 *
 *   let sx_supply = fungible_asset::supply(lp_metadata);   // == 0
 *   let ratio     = bigdecimal::from_ratio_u128(1, sx_supply);
 *                         └────── divide‑by‑zero → abort
 *
 * transaction reverts with EZeroDenominator
 */
```

---

---
### Example 3

**Auto Label:** Common vulnerability type: **Arithmetic errors due to unhandled zero divisions or improper ratio calculations, leading to lost assets, incorrect token distributions, or denied user actions.**  

**Original Text Preview:**

When providing liquidity to a pool that already has liquidity, users may lose a portion of their deposited tokens if they provide tokens in different proportions relative to the current pool reserves. While the `slippage_tolerance` parameter protects against receiving too few LP tokens, it doesn’t protect against token loss due to disproportionate deposits.

The `provide_liquidity` function in the pool manager calculates LP tokens to mint based on the minimum share ratio of provided tokens. When tokens are provided in different proportions relative to the pool’s current reserves, the excess tokens from the higher proportion are effectively donated to the pool. This way, users can lose tokens when providing liquidity with disproportionate amounts.

### Proof of Concept

First liquidity provider in a pool may provide arbitrary token amounts and set the initial price, but all other liquidity providers must provide liquidity proportionally to current pool reserves.

Here’s the relevant code from `commands.rs::provide_liquidity`:

[`commands.rs# L213-L231`](https://github.com/code-423n4/2024-11-mantra-dex/blob/26714ea59dab7ecfafca9db1138d60adcf513588/contracts/pool-manager/src/liquidity/commands.rs# L213-L231)
```

let mut asset_shares = vec![];

    for deposit in deposits.iter() {
        let asset_denom = &deposit.denom;
        let pool_asset_index = pool_assets
            .iter()
            .position(|pool_asset| &pool_asset.denom == asset_denom)
            .ok_or(ContractError::AssetMismatch)?;

        asset_shares.push(
            deposit
                .amount
                .multiply_ratio(total_share, pool_assets[pool_asset_index].amount),
        );
    }

    std::cmp::min(asset_shares[0], asset_shares[1])
}
```

Since a pool is made of two tokens and liquidity is provided in both tokens, there’s a possibility for a discrepancy: token amounts may be provided in different proportions. When this happens, the smaller of the proportions is chosen to calculate the amount of LP tokens minted.

For each deposited token, it calculates a share ratio:
```

share_ratio = deposit_amount * total_share / pool_asset_amount
```

Then it takes the minimum of these share ratios to determine LP tokens to mint:
```

final_share = min(share_ratio_token_a, share_ratio_token_b)
```

As a result, the difference in proportions will create an excess of tokens that won’t be redeemable for the amount of LP tokens minted. The excess of tokens gets, basically, donated to the pool: it’ll be shared among all liquidity providers of the pool.

While the `slippage_tolerance` argument of the `provide_liquidity` function allows liquidity providers to set the minimal amount of LP tokens they want to receive, it doesn’t allow them to minimize the disproportion of token amounts or avoid it at all.

### Recommended mitigation steps

In the `provide_liquidity` function, consider calculating optimal token amounts based on the amounts specified by user, current pool reserves, and the minimal LP tokens amount specified by user. As a reference, consider this piece from the Uniswap V2 Router: [UniswapV2Router02.sol# L45-L60](https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router02.sol# L45-L60).

**jvr0x (MANTRA) acknowledged**

**[3docSec (judge) decreased severity to Medium and commented](https://code4rena.com/audits/2024-11-mantra-dex/submissions/F-119?commentParent=iV6rmMwFBMx):**

> I consider this group a valid medium, basing on the following facts:
>
> * As said [here](https://code4rena.com/evaluate/2024-11-mantra-dex/submissions/S-212?commentParent=QgDcG5E3UD5), frontrunning is difficult to automate.
> * There is slippage protection in place that is sufficient to avoid considerable losses.
> * Because extra tokens are not returned; however, the “accidental” frontrun case can lead to non-dust value leakage, which is a solid medium severity impact.

---

---
