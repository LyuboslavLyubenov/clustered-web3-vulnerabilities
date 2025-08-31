# Cluster -1237

**Rank:** #362  
**Count:** 16  

## Label
Misuse of state variables and missing data fields lead to incorrect token mappings, flawed valuation, and ambiguous parameter tracking—compromising oracle accuracy, auditability, and system stability.

## Cluster Information
- **Total Findings:** 16

## Examples

### Example 1

**Auto Label:** Missing or premature state updates lead to inconsistent oracle data, incorrect pricing, and flawed liquidation decisions, compromising financial accuracy and system reliability in DeFi protocols.  

**Original Text Preview:**

## Set `curve.reached_migration_at`

When the curve has reached migration within buy, set `curve.reached_migration_at` to record the timestamp at which migration was reached.

## Remediation

Implement the above suggestion.

---
### Example 2

**Auto Label:** Premature exits and flawed validation lead to missed callbacks, cascading failures, and unauthorized fee drainage, compromising oracle integrity and protocol reliability.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-02-perennial-v2-3-judging/issues/6 

## Found by 
bin2chen, panprog
## Summary

Each market action requests a new oracle version which must be commited by the keepers. However, if keepers are unable to commit requested version's price (for example, no price is available at the time interval, network or keepers are down), then after a certain timeout this oracle version will be commited as invalid, using the previous valid version's price.

The issue is that when this expired oracle version is used by the market (using `oracle.at`), the version returned will be valid (`valid = true`), because oracle returns version as invalid only if `price = 0`, but the `commit` function sets the previous version's price for these, thus it's not 0.

This leads to market using invalid versions as if they're valid, keeping the orders (instead of invalidating them), which is a broken core functionality and a security risk for the protocol.

## Vulnerability Detail

When requested oracle version is commited, but is expired (commited after a certain timeout), the price of the previous valid version is set to this expired oracle version:
```solidity
function _commitRequested(OracleVersion memory version) private returns (bool) {
    if (block.timestamp <= (next() + timeout)) {
        if (!version.valid) revert KeeperOracleInvalidPriceError();
        _prices[version.timestamp] = version.price;
    } else {
        // @audit previous valid version's price is set for expired version
        _prices[version.timestamp] = _prices[_global.latestVersion]; 
    }
    _global.latestIndex++;
    return true;
}
```

Later, `Market._processOrderGlobal` reads the oracle version using the `oracle.at`, invalidating the order if the version is invalid:
```solidity
function _processOrderGlobal(
    Context memory context,
    SettlementContext memory settlementContext,
    uint256 newOrderId,
    Order memory newOrder
) private {
    OracleVersion memory oracleVersion = oracle.at(newOrder.timestamp);

    context.pending.global.sub(newOrder);
    if (!oracleVersion.valid) newOrder.invalidate();
```

However, expired oracle version will return `valid = true`, because this flag is only set to `false` if `price = 0`:
```solidity
function at(uint256 timestamp) public view returns (OracleVersion memory oracleVersion) {
    (oracleVersion.timestamp, oracleVersion.price) = (timestamp, _prices[timestamp]);
    oracleVersion.valid = !oracleVersion.price.isZero(); // @audit <<< valid = false only if price = 0
}
```

This means that `_processOrderGlobal` will treat this expired oracle version as valid and won't invalidate the order.

## Impact

Market uses invalid (expired) oracle versions as if they're valid, keeping the orders (instead of invalidating them), which is a broken core functionality and a security risk for the protocol.

## Code Snippet

`KeeperOracle._commitRequested` sets `_prices` to the last valid version's price for expired versions:
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L153-L162

`Market._processOrderGlobal` reads the oracle version using the `oracle.at`, invalidating the order if the version is invalid:
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L604-L613

`KeeperOracle.at` returns `valid = false` only if `price = 0`, but since expired version has valid price, it will be returned as a valid version:
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L109-L112

## Tool used

Manual Review

## Recommendation

Add validity map along with the price map to `KeeperOracle` when recording commited price.



## Discussion

**nevillehuang**

@arjun-io @panprog @bin2chen66 For the current supported tokens in READ.ME, I think medium severity remains appropriate given they are both stablecoins. Do you agree?

**arjun-io**

> @arjun-io @panprog @bin2chen66 For the current supported tokens in READ.ME, I think medium severity remains appropriate given they are both stablecoins. Do you agree?

I'm not entirely sure how the stablecoin in use matters here? Returning an invalid versions as valid can be very detrimental in markets where invalid versions can be triggered at will (such as in markets that close) which can result in users being able to open or close positions when they shouldn't be able to

**sherlock-admin4**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/equilibria-xyz/perennial-v2/pull/308

---
### Example 3

**Auto Label:** Missing or premature state updates lead to inconsistent oracle data, incorrect pricing, and flawed liquidation decisions, compromising financial accuracy and system reliability in DeFi protocols.  

**Original Text Preview:**

The HydraDx protocol includes an oracle. This oracle generates prices, based upon the information it receives from its sources (of which Omnipool is one). The Omnipool provides information to the oracle through the `on_liquidity_changed` and `on_trade` hooks. Whenever a trade happens or the liquidity in one of the pools changes the corresponding hooks need to be called with the updated values.

The Omnipool contract also includes the `remove_token()` function. This function can only be called by the authority and can be only called on an asset which is FROZEN and where all the liquidity shares are owned by the protocol.

```rust
ensure!(asset_state.tradable == Tradability::FROZEN, Error::<T>::AssetNotFrozen);
ensure!(
	asset_state.shares == asset_state.protocol_shares,
	Error::<T>::SharesRemaining
);
```

When the function gets called it transfers all remaining liquidity to the beneficiary and removes the token. This is a change in liquidity in the Omnipool. The functionality in terms of liquidity change is similar to the `withdraw_protocol_liquidity()` where the protocol also withdraws liquidity in the form of `protocol_shares` from the pool. When looking at the `withdraw_protocol_liquidity()` function, one can see that it calls the `on_liquidity_changed` hook at the end, so that the oracle receives the information about the liquidity change.

```rust
T::OmnipoolHooks::on_liquidity_changed(origin, info)?;
```

Unfortunately, the  `remove_token()` function does not call this hook, keeping the oracle in an outdated state. As the token is removed later on, the oracle will calculate based on liquidity that does not exist anymore in the Omnipool.

### Impact

The issue results in the oracle receiving incorrect information and calculating new prices, based on an outdated state of the Omnipool.

### Proof of Concept

The issue can be viewed when looking at the code of `remove_token()` where one can see that no call to the hook happens:

```rust
#[pallet::call_index(12)]
#[pallet::weight(<T as Config>::WeightInfo::remove_token())]
#[transactional]
pub fn remove_token(origin: OriginFor<T>, asset_id: T::AssetId, beneficiary: T::AccountId) -> DispatchResult {
	T::AuthorityOrigin::ensure_origin(origin)?;
	let asset_state = Self::load_asset_state(asset_id)?;

	// Allow only if no shares are owned by LPs and the asset is frozen.
	ensure!(asset_state.tradable == Tradability::FROZEN, Error::<T>::AssetNotFrozen);
	ensure!(
		asset_state.shares == asset_state.protocol_shares,
		Error::<T>::SharesRemaining
	);
	// Imbalance update
	let imbalance = <HubAssetImbalance<T>>::get();
	let hub_asset_liquidity = Self::get_hub_asset_balance_of_protocol_account();
	let delta_imbalance = hydra_dx_math::omnipool::calculate_delta_imbalance(
		asset_state.hub_reserve,
		I129 {
			value: imbalance.value,
			negative: imbalance.negative,
		},
		hub_asset_liquidity,
	)
	.ok_or(ArithmeticError::Overflow)?;
	Self::update_imbalance(BalanceUpdate::Increase(delta_imbalance))?;

	T::Currency::withdraw(T::HubAssetId::get(), &Self::protocol_account(), asset_state.hub_reserve)?;
	T::Currency::transfer(asset_id, &Self::protocol_account(), &beneficiary, asset_state.reserve)?;
	<Assets<T>>::remove(asset_id);
	Self::deposit_event(Event::TokenRemoved {
		asset_id,
		amount: asset_state.reserve,
		hub_withdrawn: asset_state.hub_reserve,
	});
	Ok(())
}
```

### Recommended Mitigation Steps

The issue can be mitigated by forwarding the updated asset state to the oracle by calling the `on_liquidity_changed` hook.

### Assessed type

Oracle

**[enthusiastmartin (HydraDX) disputed and commented via duplicate issue #141](https://github.com/code-423n4/2024-02-hydradx-findings/issues/141#issuecomment-1980993667):**
> The calls is not needed in mentioned functions. `sacrifice_position` does not change any liquidity and `remove_token` just removes token.

**[J4X (warden) commented](https://github.com/code-423n4/2024-02-hydradx-findings/issues/51#issuecomment-1993119034):**
 > @Lambda - This issue has been deemed as invalid due to a comment by the sponsor on Issue [#141](https://github.com/code-423n4/2024-02-hydradx-findings/issues/141). Issue #141 describes that in the functions `sacrifice_position()` and `remove_token()`, a hook call to `on_liquidity_changed` is missing. The sponsor has disputed this with the claim that in none of those functions, the liquidity gets changed, which is true for `sacrifice_position()` but not for `remove_token()`. In `sacrifice_position()`, the sacrificed positions' ownership is transferred to the protocol but the liquidity does not change. 
> 
> The same is not the case for the `remove_token()` function. As one can see in the following [code snippet](https://github.com/code-423n4/2024-02-hydradx/blob/main/HydraDX-node/pallets/omnipool/src/lib.rs#L1566), the function transfers out all liquidity that is owned by protocol shares to a beneficiary, changing the liquidity in the pool:
> 
> ```rust
> T::Currency::transfer(asset_id, &Self::protocol_account(), &beneficiary, asset_state.reserve)?;
> ```
> 
> The function documentation also [mentions](https://github.com/code-423n4/2024-02-hydradx/blob/main/HydraDX-node/pallets/omnipool/src/lib.rs#L1533C6-L1533C101) the liquidity change. 
> 
> So contrary to the comment of the sponsor, not only does the token get removed but also the liquidity changes, as the protocol-owned liquidity is sent to the beneficiary. This should result in a call to the hook so that the circuit breaker and the oracle get accordingly updated (and trigger at the right values). This could for example lead to an issue if we have a maximum liquidity change per block of 100 tokens chosen in our circuit breaker and a token gets removed with 90 tokens of protocol liquidity being withdrawn. A later call withdrawing 20 liquidity would incorrectly pass as the earlier withdrawn liquidity is not accounted for due to the missing hook call. This would undermine the security measure of the circuit breaker as the limits are not correctly enforced. Additionally, due to the missing liquidity update, the oracle will be outdated too.
> 
> I would like to mention that my issue is the only issue that fully and correctly documents the problem, as Issue #141 is reporting an invalid additional issue and also recommends an incorrect mitigation of increasing the liquidityInBlock in `sacrifice_position()`.

**[Lambda (judge) commented](https://github.com/code-423n4/2024-02-hydradx-findings/issues/51#issuecomment-1999261871):**
 > Thanks for your comment. After looking at it again, `remove_token` indeed changes the liquidity like `add_token` does. While `add_token` calls `on_liquidity_changed`, `remove_token` does not, which can lead to inconsistencies.

***

---
