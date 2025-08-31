# Cluster -1347

**Rank:** #162  
**Count:** 82  

## Label
Insufficient input validation allows invalid or zero-value parameters to pass through, leading to incorrect state updates, denial-of-service, or unintended behavior in critical financial logic.

## Cluster Information
- **Total Findings:** 82

## Examples

### Example 1

**Auto Label:** Inadequate input and state validation leads to inconsistent, exploitable system configurations and data integrity failures, enabling unauthorized state changes and financial manipulation.  

**Original Text Preview:**

In cases where the edge between tokens is not set, the `edge()` function will return a default edge. However, this default edge is not properly checked for correctness. Specifically, when `defaultEdge.amplitude == 0`, the `edge()` function should `revert` to prevent using an invalid edge, as it is used in the `SwapFacet._preSwap()` and `LiqFacet.addLiq()` functions this could lead to unexpected behavior.

The issue arises from the removal of the proper check in the `Edge.getSlot0()` function, which was never reintroduced in the `Store::edge()` function. Additionally, the removal of this check leaves the `error NoEdgeSettings(address token0, address token1);` unused in the code.

Consider adding the missing sanity check to the `Store::edge()` function.

---
### Example 2

**Auto Label:** Inadequate input and state validation leads to inconsistent, exploitable system configurations and data integrity failures, enabling unauthorized state changes and financial manipulation.  

**Original Text Preview:**

**Severity:** Informational

**Path:** circom/components.circom, circom/mint.circom, circom/transfer.circom, circom/withdraw.circom

**Description:** Less Than comparison circuits ensure values are below the BabyJubJub subgroup order and validate that spending does not exceed the sender’s balance, preserving arithmetic integrity and transactional correctness. There can be applied some optimizations in the comparison:
```
    var baseOrder = 2736030358979909402780800718157159386076813972158567259200215660948447373041;

    component bitCheck1 = Num2Bits(252);
    bitCheck1.in <== random;

    component bitCheck2 = Num2Bits(252);
    bitCheck2.in <== baseOrder;

    component lt = LessThan(252);
    lt.in[0] <== random;
    lt.in[1] <== baseOrder;
    lt.out === 1;
```
Since `baseOrder` is a constant, the `bitCheck2` operation is redundant because the constraints are already preserved.
```
    component bitCheck3 = Num2Bits(252);
    bitCheck3.in <== SenderBalance + 1;

    component checkValue = LessThan(252);
    checkValue.in[0] <== ValueToTransfer;
    checkValue.in[1] <== SenderBalance + 1;
    checkValue.out === 1;
```
In the value comparison section, the `LessThan` component can be replaced with `LessEqThan`, and `SenderBalance + 1` should be updated to `SenderBalance`.


**Remediation:**  Apply the optimizations in PoseidonDecrypt, CheckPublicKey, CheckValue, CheckPCT, MintCircuit, TransferCircuit, WithdrawCircuit components.

**Status:**  Acknowledged

- - -

---
### Example 3

**Auto Label:** Inadequate input and state validation leads to inconsistent, exploitable system configurations and data integrity failures, enabling unauthorized state changes and financial manipulation.  

**Original Text Preview:**

## Diﬃculty: Low

## Type: Data Validation

**File Path:** `holders-contracts/packages/ton/contracts/ton-card-v8.fc`

### Description
The `ctx_key_updates` storage variable contains the public key used to verify the signature on code updates. There is no clear specification about how this key is generated and how the associated private key is stored or protected in the back end.

If the private key is leaked, anybody can sign arbitrary updates to card contracts and change their functionality. There is currently no way to prevent this scenario, as the public key is constant and immutable in all card contracts, and the only requirement for updating a contract’s code is a message from the user.

```cpp
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

**Figure 8.1:** Signature verification using the hash of the code cell (holders-contracts/packages/ton/contracts/ton-card-v8.fc#L673–L693)

### Recommendations
- **Short term:** Allow a trusted actor to update the public key used to verify signatures in case they have to be changed.
- **Long term:** Consider on-chain alternatives for signature verification, such as a variation of the verification scheme recommended in the long-term recommendations for issue TOB-WHC-6. Implement industry-standard data verification algorithms and processes, avoiding single points of failure for critical validations.

---
