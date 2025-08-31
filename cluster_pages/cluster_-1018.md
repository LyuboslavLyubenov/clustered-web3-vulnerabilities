# Cluster -1018

**Rank:** #70  
**Count:** 244  

## Label
Insufficient input validation and improper state initialization lead to out-of-bounds access, undefined behavior, and unauthorized state manipulation, enabling exploitation through invalid data, race conditions, or inconsistent transitions.

## Cluster Information
- **Total Findings:** 244

## Examples

### Example 1

**Auto Label:** Insufficient state validation leads to persistent or incorrect access control, enabling unauthorized interactions, asset draining, or invalid operations.  

**Original Text Preview:**

In `ZipperFactoryVault:createVault`, two mapping values, `tokenToVault` and `vaultToToken`, are assigned. The issue arises when a changer updates these values later using a new vault address. However, the data for the `oldVault` is not deleted, leading to potential issues with outdated mappings.

---
### Example 2

**Auto Label:** Insufficient state validation leads to persistent or incorrect access control, enabling unauthorized interactions, asset draining, or invalid operations.  

**Original Text Preview:**

Each vault must be unique to a token. However, the `createVault` and `changeVault` functions do not check whether the vault key is already assigned to another token. This could result in a vault being overwritten or mistakenly reassigned, violating the uniqueness constraint.

Mitigation:
Add the following check to both functions to ensure the vault is not already in use by another token:

```solidity
require(vaultToToken[chainId][vault].length == 0, "Vault already exists for another token");
```

---
### Example 3

**Auto Label:** Missing state validation allows unauthorized or invalid transitions, leading to inconsistent, unsafe state changes and potential misuse of access controls or data integrity.  

**Original Text Preview:**

<https://github.com/code-423n4/2025-03-starknet/blob/512889bd5956243c00fc3291a69c3479008a1c8a/workspace/apps/perpetuals/contracts/src/core/core.cairo# L959-L988>

### Root Cause

The `_execute_transfer` function applies a state change (`apply_diff`) to the sender’s position before validating its health (`_validate_healthy_or_healthier_position`). This results in the potential application of the state change a second time during validation, which can lead to failure if the sender’s position becomes unhealthy after the second state change.

### Impact

* **Inconsistent State**: The sender’s position may be healthy, but two times applying the diff could make the `_validate_healthy_or_healthier_position` to revert.

### Recommended Mitigation Steps

To ensure the operations are executed in the correct order, make the following change.

**Updated Code**
```

fn _execute_transfer(
    ref self: ContractState,
    recipient: PositionId,
    position_id: PositionId,
    collateral_id: AssetId,
    amount: u64,
) {
    let position_diff_sender = PositionDiff { collateral_diff: -amount.into(), synthetic_diff: Option::None };
    let position_diff_recipient = PositionDiff { collateral_diff: amount.into(), synthetic_diff: Option::None };

+   self._validate_healthy_or_healthier_position(
+       position_id: position_id,
+       position: self.positions.get_position_snapshot(position_id),
+       position_diff: position_diff_sender
+   );

    self.positions.apply_diff(position_id: position_id, position_diff: position_diff_sender);
    self.positions.apply_diff(position_id: recipient, position_diff: position_diff_recipient);

    let position = self.positions.get_position_snapshot(position_id);
-   self._validate_healthy_or_healthier_position(
-       position_id: position_id,
-       position: position,
-       position_diff: position_diff_sender
-   );
}
```

**Steps:**

1. **Validate sender’s position health** before applying any state changes.
2. **Apply diffs** only if the validation passes to ensure the sender’s position remains healthy.
3. Test the implementation with both success and failure cases to confirm the behavior works as expected.

---

This version provides a concise explanation of the issue, impact, and recommended solution. The steps are clearly laid out for better actionability. Let me know if you need further adjustments!

### Proof of Concept
```

#[test]
fn test_successful_trade() {
    // Setup state, token and user:
    let cfg: PerpetualsInitConfig = Default::default();
    let token_state = cfg.collateral_cfg.token_cfg.deploy();
    let mut state = setup_state_with_active_asset(cfg: @cfg, token_state: @token_state);

    let user_a = Default::default();
    init_position(cfg: @cfg, ref :state, user: user_a);

    let user_b = UserTrait::new(position_id: POSITION_ID_2, key_pair: KEY_PAIR_2());
    init_position(cfg: @cfg, ref :state, user: user_b);

    // Test params:
    let BASE = -10;
    let QUOTE = 75;
    let FEE = 1;

    // Setup parameters:
    let expiration = Time::now().add(delta: Time::days(1));

    let collateral_id = cfg.collateral_cfg.collateral_id;
    let synthetic_id = cfg.synthetic_cfg.synthetic_id;

    let order_a = Order {
        position_id: user_a.position_id,
        salt: user_a.salt_counter,
        base_asset_id: synthetic_id,
        base_amount: BASE,
        quote_asset_id: collateral_id,
        quote_amount: QUOTE,
        fee_asset_id: collateral_id,
        fee_amount: FEE,
        expiration,
    };

    let order_b = Order {
        position_id: user_b.position_id,
        base_asset_id: synthetic_id,
        base_amount: -BASE,
        quote_asset_id: collateral_id,
        quote_amount: -QUOTE,
        fee_asset_id: collateral_id,
        fee_amount: FEE,
        expiration,
        salt: user_b.salt_counter,
    };

    let hash_a = order_a.get_message_hash(user_a.get_public_key());
    let hash_b = order_b.get_message_hash(user_b.get_public_key());
    let signature_a = user_a.sign_message(hash_a);
    let signature_b = user_b.sign_message(hash_b);
    let operator_nonce = state.get_operator_nonce();

    let mut spy = snforge_std::spy_events();
    // Test:
    cheat_caller_address_once(contract_address: test_address(), caller_address: cfg.operator);
    state
        .trade(
            :operator_nonce,
            :signature_a,
            :signature_b,
            :order_a,
            :order_b,
            actual_amount_base_a: BASE,
            actual_amount_quote_a: QUOTE,
            actual_fee_a: FEE,
            actual_fee_b: FEE,
        );

    // Catch the event.
    let events = spy.get_events().emitted_by(test_address()).events;
    assert_trade_event_with_expected(
        spied_event: events[0],
        order_a_position_id: user_a.position_id,
        order_a_base_asset_id: synthetic_id,
        order_a_base_amount: BASE,
        order_a_quote_asset_id: collateral_id,
        order_a_quote_amount: QUOTE,
        fee_a_asset_id: collateral_id,
        fee_a_amount: FEE,
        order_b_position_id: user_b.position_id,
        order_b_base_asset_id: synthetic_id,
        order_b_base_amount: -BASE,
        order_b_quote_asset_id: collateral_id,
        order_b_quote_amount: -QUOTE,
        fee_b_asset_id: collateral_id,
        fee_b_amount: FEE,
        actual_amount_base_a: BASE,
        actual_amount_quote_a: QUOTE,
        actual_fee_a: FEE,
        actual_fee_b: FEE,
        order_a_hash: hash_a,
        order_b_hash: hash_b,
    );

    // Check:
    let position_a = state.positions.get_position_snapshot(position_id: user_a.position_id);
    let user_a_collateral_balance = state
        .positions
        .get_collateral_provisional_balance(position: position_a);
    let user_a_synthetic_balance = state
        .positions
        .get_synthetic_balance(position: position_a, :synthetic_id);

let position_b = state.positions.get_position_snapshot(position_id: user_b.position_id);
    let user_b_collateral_balance = state
        .positions
        .get_collateral_provisional_balance(position: position_b);
    let user_b_synthetic_balance = state
        .positions
        .get_synthetic_balance(position: position_b, :synthetic_id);

let position = state.positions.get_position_snapshot(position_id: FEE_POSITION);
    let fee_position_balance = state.positions.get_collateral_provisional_balance(:position);
    assert!(fee_position_balance == (FEE + FEE).into());

    let expiration = Time::now().add(delta: Time::days(1));
    let collateral_id = cfg.collateral_cfg.collateral_id;
    let operator_nonce = state.get_operator_nonce();

    let transfer_args = TransferArgs {
        position_id: user_a.position_id,
        recipient: user_b.position_id,
        salt: user_a.salt_counter,
        expiration: expiration,
        collateral_id,
        amount: 1500,
    };

    let mut spy = snforge_std::spy_events();
    let msg_hash = transfer_args.get_message_hash(user_a.get_public_key());
    let sender_signature = user_a.sign_message(msg_hash);
    // Test:
    cheat_caller_address_once(contract_address: test_address(), caller_address: user_a.address);
    state
        .transfer_request(
            signature: sender_signature,
            recipient: transfer_args.recipient,
            position_id: transfer_args.position_id,
            amount: transfer_args.amount,
            expiration: transfer_args.expiration,
            salt: transfer_args.salt,
        );
    cheat_caller_address_once(contract_address: test_address(), caller_address: cfg.operator);
    state
        .transfer(
            :operator_nonce,
            recipient: transfer_args.recipient,
            position_id: transfer_args.position_id,
            amount: transfer_args.amount,
            expiration: transfer_args.expiration,
            salt: transfer_args.salt,
        );

    // Catch the event.
    let events = spy.get_events().emitted_by(test_address()).events;
    assert_transfer_request_event_with_expected(
        spied_event: events[0],
        position_id: transfer_args.position_id,
        recipient: transfer_args.recipient,
        collateral_id: transfer_args.collateral_id,
        amount: transfer_args.amount,
        expiration: transfer_args.expiration,
        transfer_request_hash: msg_hash,
    );
    assert_transfer_event_with_expected(
        spied_event: events[1],
        position_id: transfer_args.position_id,
        recipient: transfer_args.recipient,
        collateral_id: transfer_args.collateral_id,
        amount: transfer_args.amount,
        expiration: transfer_args.expiration,
        transfer_request_hash: msg_hash,
    );

    // Check:
    let sender_position = state.positions.get_position_snapshot(position_id: user_a.position_id);
    let sender_collateral_balance = state
        .positions
        .get_collateral_provisional_balance(position: sender_position);
    //assert!(sender_collateral_balance == COLLATERAL_BALANCE_AMOUNT.into() - TRANSFER_AMOUNT.into());

    let recipient_position = state
        .positions
        .get_position_snapshot(position_id: user_b.position_id);
    let recipient_collateral_balance = state
        .positions
        .get_collateral_provisional_balance(position: recipient_position);

}
```

**[oded (Starknet Perpetual) confirmed](https://code4rena.com/audits/2025-03-starknet-perpetual/submissions/F-33?commentParent=NfXPxvYUyMT)**

Code4rena judging staff adjusted the severity of Finding [H-01], after reviewing additional context provided by the sponsor.

---

---
