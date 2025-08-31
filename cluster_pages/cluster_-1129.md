# Cluster -1129

**Rank:** #177  
**Count:** 69  

## Label
Price-based front-running and immutable bounds lead to permanent collateral locking or unfair liquidation incentives during market volatility or oracle delays.

## Cluster Information
- **Total Findings:** 69

## Examples

### Example 1

**Auto Label:** **Oracle price validation failures leading to incorrect pricing, false liquidations, and exploitable market misalignments.**  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Medium

## Description

`OracleManager::generatePerformance` is supposed to be called once every hour (or at any other interval specified by `MIN_UPDATE_INTERVAL`). It updates the stats of every validator and also updates the `totalBalance`. If there are any rewards or slashing amounts, they are supposed to be accounted for and `ValidatorManager` should be updated accordingly.
```solidity
    function generatePerformance() external whenNotPaused onlyRole(OPERATOR_ROLE) returns (bool) {
        // ..

        // Update validators with averaged values
        for (uint256 i = 0; i < validatorCount; i++) {
            // ...

            // Handle slashing
            if (avgSlashAmount > previousSlashing) {
                uint256 newSlashAmount = avgSlashAmount - previousSlashing;
@>                validatorManager.reportSlashingEvent(validator, newSlashAmount);
            }

            // ...
        }

        // ...

        return true;
    }
```
As we can see, if the accumulated slashing amount for this particular validator has increased, `reportSlashingEvent` will be triggered with the increase passed as a parameter.
```solidity
    function reportSlashingEvent(address validator, uint256 amount)
       // ...
    {
        require(amount > 0, "Invalid slash amount");

        Validator storage val = _validators[_validatorIndexes.get(validator)];
@>        require(val.balance >= amount, "Insufficient stake for slashing");

        // Update balances
        unchecked {
            // These operations cannot overflow:
            // - val.balance >= amount (checked above)
            // - totalBalance >= val.balance (invariant maintained by the contract)
            val.balance -= amount;
            totalBalance -= amount;
        }

       // ...
    }
```
As shown in the highlighted section, the entire `reportSlashingEvent` (and therefore `generatePerformance` as well) will revert if the new slashing amount is greater than the validator’s previously reported balance. However, this is actually a very likely scenario and could easily happen due to the intervals between updates.

For example, at time `T`, the validator’s balance could be `100`. At `T + 1 hour`, the validator’s balance could grow to `500`, and the `slashingAmount` could be `110`. Due to this bug, the code will require `oldBalance > newSlashingAmount`, which would revert since `slashingAmount` is `110` and `oldBalance` is only `100`. Situations like this are especially likely to occur in the first few days after a validator is activated.

## Recommendations

Consider implementing a mechanism for handling slashing amounts similar to how rewards are handled, or compare the new slashing amount with the validator’s actual up to-date-balance.

---
### Example 2

**Auto Label:** **Oracle price validation failures leading to incorrect pricing, false liquidations, and exploitable market misalignments.**  

**Original Text Preview:**

In the function `getWTAOByrsTAOAfterFee()` code should calculate wTAO amount and then subtract the `unstakingFee` from it. This won't cause an issue in the current code because the ratio is 1 but in general the formula is wrong. The issue exists in COMAI contracts too.

---
### Example 3

**Auto Label:** **Oracle price validation failures leading to incorrect pricing, false liquidations, and exploitable market misalignments.**  

**Original Text Preview:**

<https://github.com/code-423n4/2025-03-starknet/blob/512889bd5956243c00fc3291a69c3479008a1c8a/workspace/apps/perpetuals/contracts/src/core/components/assets/assets.cairo# L109-L145>

<https://github.com/code-423n4/2025-03-starknet/blob/512889bd5956243c00fc3291a69c3479008a1c8a/workspace/apps/perpetuals/contracts/src/core/components/assets/assets.cairo# L350>

<https://github.com/code-423n4/2025-03-starknet/blob/512889bd5956243c00fc3291a69c3479008a1c8a/workspace/apps/perpetuals/contracts/src/core/components/assets/assets.cairo# L708>

<https://github.com/code-423n4/2025-03-starknet/blob/512889bd5956243c00fc3291a69c3479008a1c8a/workspace/apps/perpetuals/contracts/src/core/components/assets/assets.cairo# L746-L762>

### Finding description and impact

An oracle is added for a synthetic asset through the governance protected function `assets.add_oracle_to_asset()`. The oracle is saved for a particular asset in the `asset_oracle` storage by mapping its public key to the asset name + oracle name.
```

fn add_oracle_to_asset(
    ref self: ComponentState<TContractState>,
    asset_id: AssetId,
    oracle_public_key: PublicKey,
    oracle_name: felt252,
    asset_name: felt252,
) {
    // ...
    // Validate the oracle does not exist.
    let asset_oracle_entry = self.asset_oracle.entry(asset_id).entry(oracle_public_key);
    let asset_oracle_data = asset_oracle_entry.read();
    assert(asset_oracle_data.is_zero(), ORACLE_ALREADY_EXISTS);
    // ...
    // Add the oracle to the asset.
    let shifted_asset_name = TWO_POW_40.into() * asset_name;
    asset_oracle_entry.write(shifted_asset_name + oracle_name);
    // ...
}
```

The function `assets.price_tick()` updates the price of an asset where a list of `signed_prices` is supplied that must only contain prices that were signed by oracles that were added through `assets.

`add_oracle_to_asset()` for that asset. The validation of the list is done in `assets._validate_price_tick()` where `assets._validate_oracle_signature()` is called for each signed price. This function attempts to read from storage the packed asset and oracle names stored against the supplied `signed_price.signer_public_key`, hash the read value with the supplied oracle price and timestamp and validate if the signature supplied for that hash value corresponds to the supplied public key.

However, there is no validation if the supplied `signed_price.signer_public_key` is an existing key in storage. For an arbitrary signer key, the `self.asset_oracle.entry(asset_id).read(signed_price.signer_public_key)` operation returns an empty `packed_asset_oracle` instead of a panic halting execution. This allows for an arbitrary `signed_price.signer_public_key` to create a signature over packed asset and oracle names that are 0 and bypass `validate_oracle_siganture()`, therefore, supplying an arbitrary price without the signer key being approved and added by the governance admin through `add_oracle_to_asset()`.
```

fn _validate_oracle_signature(
    self: @ComponentState<TContractState>, asset_id: AssetId, signed_price: SignedPrice,
) {
    // @audit won't panic on non existing signer_price.signer_public_key
    let packed_asset_oracle = self
        .asset_oracle
        .entry(asset_id)
        .read(signed_price.signer_public_key);
    let packed_price_timestamp: felt252 = signed_price.oracle_price.into()
        * TWO_POW_32.into()
        + signed_price.timestamp.into();
    let msg_hash = core::pedersen::pedersen(packed_asset_oracle, packed_price_timestamp);
    validate_stark_signature(
        public_key: signed_price.signer_public_key,
        :msg_hash,
        signature: signed_price.signature,
    );
}
```

### Recommended mitigation steps

Panic if the supplied `signed_price.signer_public_key` maps to an empty packed oracle name + asset name.

### Proof of Concept

* Place the modified `test_price_tick_basic()` in `test_core.cairo`
* Execute with `scarb test test_price_tick_basic`
* The test shows how an invalid oracle can provide signature for `price_tick()`
```

 fn test_price_tick_basic() {
     let cfg: PerpetualsInitConfig = Default::default();
     let token_state = cfg.collateral_cfg.token_cfg.deploy();
     let mut state = setup_state_with_pending_asset(cfg: @cfg, token_state: @token_state);
     let mut spy = snforge_std::spy_events();
     let asset_name = 'ASSET_NAME';
     let oracle1_name = 'ORCL1';
     let oracle1 = Oracle { oracle_name: oracle1_name, asset_name, key_pair: KEY_PAIR_1() };
     let synthetic_id = cfg.synthetic_cfg.synthetic_id;
     cheat_caller_address_once(contract_address: test_address(), caller_address: cfg.app_governor);
     state
         .add_oracle_to_asset(
             asset_id: synthetic_id,
             oracle_public_key: oracle1.key_pair.public_key,
             oracle_name: oracle1_name,
             :asset_name,
         );
     let old_time: u64 = Time::now().into();
     let new_time = Time::now().add(delta: MAX_ORACLE_PRICE_VALIDITY);
     assert!(state.assets.get_num_of_active_synthetic_assets() == 0);
     start_cheat_block_timestamp_global(block_timestamp: new_time.into());
     cheat_caller_address_once(contract_address: test_address(), caller_address: cfg.operator);
-    let oracle_price: u128 = ORACLE_PRICE;
+    // @audit can set whatever price here
+    let oracle_price: u128 = ORACLE_PRICE*1000;
     let operator_nonce = state.get_operator_nonce();
+
+    // @audit use key pair 3 even though the public key hasn't been added through add_oracle_to_asset()
+    let malicious_oracle_signer = Oracle {oracle_name: '', asset_name: '', key_pair: KEY_PAIR_3()};
+
     state
         .price_tick(
             :operator_nonce,
             asset_id: synthetic_id,
             :oracle_price,
+            // @audit invalid oracle
             signed_prices: [
-                oracle1.get_signed_price(:oracle_price, timestamp: old_time.try_into().unwrap())
+                malicious_oracle_signer.get_signed_price(:oracle_price, timestamp: old_time.try_into().unwrap())
             ]
                 .span(),
         );

     // Catch the event.
     let events = spy.get_events().emitted_by(test_address()).events;
     assert_add_oracle_event_with_expected(
         spied_event: events[0],
         asset_id: synthetic_id,
         :asset_name,
         oracle_public_key: oracle1.key_pair.public_key,
         oracle_name: oracle1_name,
     );
     assert_asset_activated_event_with_expected(spied_event: events[1], asset_id: synthetic_id);
     assert_price_tick_event_with_expected(
-        spied_event: events[2], asset_id: synthetic_id, price: PriceTrait::new(value: 100),
+        spied_event: events[2], asset_id: synthetic_id, price: PriceTrait::new(value: 100_000),
     );

     assert!(state.assets.get_synthetic_config(synthetic_id).status == AssetStatus::ACTIVE);
     assert!(state.assets.get_num_of_active_synthetic_assets() == 1);

     let data = state.assets.get_synthetic_timely_data(synthetic_id);
     assert!(data.last_price_update == new_time);
-    assert!(data.price.value() == 100 * PRICE_SCALE);
+    assert!(data.price.value() == 100_000 * PRICE_SCALE);
 }
```

**[oded (Starknet Perpetual) confirmed](https://code4rena.com/audits/2025-03-starknet-perpetual/submissions/F-31?commentParent=WQLjyuZFse5)**

---

---
