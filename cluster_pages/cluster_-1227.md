# Cluster -1227

**Rank:** #409  
**Count:** 11  

## Label
**Bounds checking failures leading to invalid state, memory corruption, or unauthorized access due to missing runtime validation of owner limits and thresholds.**

## Cluster Information
- **Total Findings:** 11

## Examples

### Example 1

**Auto Label:** Missing runtime bounds validation leading to potential overflow, out-of-range state changes, or unintended resource exhaustion via unchecked parameter values.  

**Original Text Preview:**

##### Description
There is a `maxPositiveTokenRebase` parameter check at the [following line](https://github.com/lidofinance/core/blob/efeff81c18f85451ebf98e8fd8bb78b8eb0095f6/contracts/0.8.9/sanity_checks/OracleReportSanityChecker.sol#L880). This check allows `maxPositiveTokenRebase` to be assigned a value between `1` and `type(uint64).max`. But `maxPositiveTokenRebase` should be in range from `1` to `1e9` or equal to `type(uint64).max`.

##### Recommendation
We recommend adding a more strict check for the `maxPositiveTokenRebase` parameter.

---
### Example 2

**Auto Label:** **Bounds checking failures leading to invalid state, memory corruption, or unauthorized access due to missing runtime validation of owner limits and thresholds.**  

**Original Text Preview:**

## MSafe Owners Limit and Implementation Details

The maximum limit for the number of owners of an msafe is derived from the `aptos-core` multisig crate’s `MAX_NUMBER_OF_PUBLIC_KEYS`. This semantic dependency should be noted.

## Source Code Reference

Path: `aptos-core/aptos-move/framework/aptos-stdlib/sources/cryptography/multied25519.move`

```rust
const BITMAP_NUM_OF_BYTES: u64 = 4;

/// Max number of ed25519 public keys allowed in multi-ed25519 keys
const MAX_NUMBER_OF_PUBLIC_KEYS: u64 = 32;
```

Since a nonce public key is added while deriving the authentication key, the values will differ by 1. This should be mentioned in the comments while defining the `MAX_MSAFE_OWNERS_LIMIT`.

## Remediation

Add a comment stating that the const `MAX_MSAFE_OWNERS_LIMIT` is derived from the Aptos `MAX_NUMBER_OF_PUBLIC_KEYS`.

## Code Diff

```diff
--- a/move/sources/creator.move
+++ b/move/sources/creator.move
@@ -146,7 +146,10 @@ module msafe::creator {
 /// the real-time network situation.
 const MIN_REGISTER_TX_GAS: u64 = 2000;
 const MIN_REGISTER_TX_GAS_PRICE: u64 = 1;
+
+ /// Derived constant from aptos core multi-sign implementation
+ /// The allowed max in aptos is 32, since we add a custom nonce
+ /// this is off by one to the original constant defined in the aptos
 const MAX_MSAFE_OWNERS_LIMIT: u64 = 31;
```

## Patch

Fixed in commit **921ad74**.

---
### Example 3

**Auto Label:** **Bounds checking failures leading to invalid state, memory corruption, or unauthorized access due to missing runtime validation of owner limits and thresholds.**  

**Original Text Preview:**

## Vulnerability Report

An additional nonce key is added in `derive_multisig_auth_key`. This means that the actual `MAX_MSAFE_OWNERS_LIMIT` should be one less than the Aptos enforced maximum of 32, or a total of 31.

## Code Snippet

```move
move/sources/utils.move RUST
public fun derive_multisig_auth_key(
    pubkeys: vector<vector<u8>>,
    threshold: u8,
    nonce: u64,
    module_address: address
): vector<u8> {
    // Write the owner public keys to buffer.
    let pk_bytes = vector::empty<u8>();
    let i = 0;
    while (i < vector::length(&pubkeys)) {
        vector::append(&mut pk_bytes, *vector::borrow(&pubkeys, i));
        i = i + 1;
    };
    // Write the additional nonce as the last public key.
    vector::append(&mut pk_bytes, nonce_to_public_key(nonce, module_address));
    // Write signature threshold and multi-ed25519 schema identifier.
    vector::push_back(&mut pk_bytes, threshold);
    vector::push_back(&mut pk_bytes, 1);
    // Derive the authentication key from the buffer.
    hash::sha3_256(pk_bytes)
}
```

## Remediation

Change the `MAX_MSAFE_OWNERS_LIMIT` from 32 to 31.

## DIFF

```diff
--- a/move/sources/creator.move
+++ b/move/sources/creator.move
Momentum Safe Audit 04 | Vulnerabilities
@@ -146,7 +146,7 @@ module msafe::creator {
 /// the real-time network situation.
 const MIN_REGISTER_TX_GAS: u64 = 2000;
 const MIN_REGISTER_TX_GAS_PRICE: u64 = 1;
- const MAX_MSAFE_OWNERS_LIMIT: u64 = 32;
+ const MAX_MSAFE_OWNERS_LIMIT: u64 = 31;
 /// Wallet creation information stored under the deployer's account.
 struct PendingMultiSigCreations has key, drop {
```

## Patch

Fixed in commit `353c75e`.

---
