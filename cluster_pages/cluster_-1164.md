# Cluster -1164

**Rank:** #193  
**Count:** 59  

## Label
Unauthorized, stealthy contract upgrades bypassing immutability, validation, or time locks, enabling malicious state changes, backdoor access, or disruption of core functionality without user notice or auditability.

## Cluster Information
- **Total Findings:** 59

## Examples

### Example 1

**Auto Label:** Unauthorized, stealthy contract upgrades bypassing immutability, validation, or time locks, enabling malicious state changes, backdoor access, or disruption of core functionality without user notice or auditability.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

`set_strategy` can be called by governance to change the list of an adapter's strategies. This could incorporate a new adapter or change the previous adapter's ratio. However, it will incorrectly remove the `last_asset_value` of the adapter's strategy if the new strategy incorporates a previously used adapter.

```python
def _set_strategy(_proposer: address, _strategies : AdapterStrategy[MAX_ADAPTERS], _min_proposer_payout : uint256, pregen_info: DynArray[Bytes[4096], MAX_ADAPTERS]) -> bool:
    assert msg.sender == self.governance, "Only Governance DAO may set a new strategy."
    assert _proposer != empty(address), "Proposer can't be null address."

    # Are we replacing the old proposer?
    if self.current_proposer != _proposer:

        current_assets : uint256 = self._totalAssetsNoCache() # self._totalAssetsCached()

        # Is there enough payout to actually do a transaction?
        yield_fees : uint256 = 0
        strat_fees : uint256 = 0
        yield_fees, strat_fees = self._claimable_fees_available(current_assets)
        if strat_fees > self.min_proposer_payout:

            # Pay prior proposer his earned fees.
            self._claim_fees(FeeType.PROPOSER, 0, pregen_info, current_assets)

        self.current_proposer = _proposer
        self.min_proposer_payout = _min_proposer_payout

    # Clear out all existing ratio allocations.
    for adapter in self.adapters:
        self.strategy[adapter] = empty(AdapterValue)

    # Now set strategies according to the new plan.
    for strategy in _strategies:
        plan : AdapterValue = empty(AdapterValue)
        plan.ratio = strategy.ratio
        # @audit - it will clear `last_asset_value` if it previously exist
>>>     self.strategy[strategy.adapter] = plan

    log StrategyActivation(_strategies, _proposer)

    return True
```

`last_asset_value` is a safeguard/protection implemented in case of sudden loss or if the LP is compromised. Setting the value to 0 will remove and bypass the protection once rebalancing happens.

## Recommendations

Set `last_asset_value` to the previous value if it previously exists.

---
### Example 2

**Auto Label:** Unauthorized, stealthy contract upgrades bypassing immutability, validation, or time locks, enabling malicious state changes, backdoor access, or disruption of core functionality without user notice or auditability.  

**Original Text Preview:**

## Diﬃculty: High

## Type: Data Validation

**File Path:** `holders-contracts/packages/ton/contracts/ton-card-v8.fc`

### Description
Users are able to update the code of their jetton cards and TON cards to any version that was ever released, including older versions. Additionally, if the public key used for signing is the same for both contracts, a jetton card can be upgraded to a TON card and vice versa.

The `op::update` operation allows a user to update the code of their contract if the new contract code was hashed and signed by the stored key in `ctx_key_updates`. This operation is shown in figure 6.1:

```c
if (op == op::update) {
    throw_unless(error::not_authorized, equal_slice_bits(sender_address, ctx_address_a)); // throw if not user
    throw_if(error::invalid_message, msg_value < 50000000); // 0.05 TON
    var query_id = in_msg_body~load_uint(64);
    var code = in_msg_body~load_ref();
    // Check signature validity
    var signature = in_msg_body~load_ref().begin_parse();
    throw_unless(error::invalid_message, signature.slice_bits() == 512);
    // Check hash
    var hash = cell_hash(code);
    throw_unless(error::not_authorized, check_signature(hash, signature, ctx_key_updates)); // verify signature
    // Update code
    set_code(code);
    do_send_message(sender_address, send_mode::carry_remaining_value, response::update, query_id, 0, in_msg_body);
    return ();
}
```

*Figure 6.1: The `op::update` operation in the jetton-card-v3 contract (holders-contracts/packages/ton/contracts/jetton-card-v3.fc#L871–L891)*

However, this operation does not prevent the code from being updated to an older version of the contract. Additionally, if the `ctx_key_updates` state variable is the same in the jetton-card-v3 and ton-card-v8 contracts, users can update their cards to a different type of card, which will completely and permanently break the cards’ functionality.

### Exploit Scenario
Alice sees a new update for the ton-card-v8 contract but mistakenly submits it to her jetton card. Since the `ctx_key_updates` public key is the same for both the jetton-card-v3 and ton-card-v8 contracts, the update passes, but Alice’s jetton card becomes permanently nonfunctional.

### Recommendations
- **Short term:** Ensure that a unique key is used for the ton-card-v8 and jetton-card-v3 contracts. Consider disabling code version downgrade in order to prevent users from exploiting known issues of older code versions.
- **Long term:** Consider implementing a two-step update procedure that requires an external validation of the update hash, in order to avoid bricking the contracts. In this scenario, users would propose the update, a message will be sent to this external actor to verify the hash and wallet type, and if it is valid, a new message accepting the update will be sent to the card contract.

---
### Example 3

**Auto Label:** Unauthorized, stealthy contract upgrades bypassing immutability, validation, or time locks, enabling malicious state changes, backdoor access, or disruption of core functionality without user notice or auditability.  

**Original Text Preview:**

##### Description

The NEAR blockchain allows for redeploying a contract to the same Account ID to modify its code. However, basic redeployment can cause issues if the contract's state requires changes, such as alterations in variable types or the addition or removal of storage variables. In such cases, upgrading via code redeployment can prevent the successful deserialization of the state, rendering the contract inoperable.

  

Notably, the Atlas and atBTC contracts do not include a migration feature, making it impossible to upgrade their states as needed.

##### BVSS

[AO:S/AC:L/AX:M/R:N/S:C/C:N/A:C/I:C/D:C/Y:N (2.5)](/bvss?q=AO:S/AC:L/AX:M/R:N/S:C/C:N/A:C/I:C/D:C/Y:N)

##### Recommendation

It is recommended to implement a state-updating upgrade functionality to mitigate the potential limitations that may arise from a code upgrade using full access keys.

##### Remediation

**SOLVED**: The **Atlas Protocol team** solved the issue in the specified commit id.

##### Remediation Hash

<https://github.com/atlasprotocol-com/atlasprotocol/commit/9747316da0ee219516a42b702c6fc11aeb3d8546>

---
