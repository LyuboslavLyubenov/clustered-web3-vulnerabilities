# Cluster -1334

**Rank:** #411  
**Count:** 11  

## Label
Failure to validate message origin or target leads to unauthorized fund transfers, double-spending, or permanent loss of assets due to insufficient cross-domain authentication and state consistency checks.

## Cluster Information
- **Total Findings:** 11

## Examples

### Example 1

**Auto Label:** Failure to validate payload fields and signer consistency leads to acceptance of invalid or unsupported transactions, causing incorrect state transitions and denial of service.  

**Original Text Preview:**

In `transaction_signing.go`, there are three functions responsible for returning the appropriate Signer type for the chain:[`LatestSignerForChainID`](https://github.com/mantlenetworkio/op-geth/blob/4a20fa61a79e4c4cd61138e467332368d7fc8a8d/core/types/transaction_signing.go#L86-L91),[`LatestSigner`](https://github.com/mantlenetworkio/op-geth/blob/4a20fa61a79e4c4cd61138e467332368d7fc8a8d/core/types/transaction_signing.go#L64-L77), and [`MakeSigner`](https://github.com/mantlenetworkio/op-geth/blob/4a20fa61a79e4c4cd61138e467332368d7fc8a8d/core/types/transaction_signing.go#L40-L55). While these functions differ in their inputs, they are expected to return the same Signer types.

* `LatestSignerForChainID`: Only accepts the chain ID as an argument.
* `LatestSigner`: Accepts the chain configuration.
* `MakeSigner`: Accepts both the chain configuration and the block number.

However, there is an inconsistency. The `LatestSignerForChainID` function returns [`cancunSigner`](https://github.com/mantlenetworkio/op-geth/blob/4a20fa61a79e4c4cd61138e467332368d7fc8a8d/core/types/transaction_signing.go#L173), reflecting that the Cancun upgrade is expected to be the most recent op-geth update adopted by Mantle. In contrast, the other two functions, even if the chain has been upgraded to Cancun, return [`londonSigner`](https://github.com/mantlenetworkio/op-geth/blob/4a20fa61a79e4c4cd61138e467332368d7fc8a8d/core/types/transaction_signing.go#L242) instead of `cancunSigner`.

The only difference between [`londonSigner`](https://github.com/ethereum-optimism/op-geth/blob/831e3fd7447f32be6a6d8e06b674479c400e7416/core/types/transaction_signing.go#L248-L252) and [`cancunSigner`](https://github.com/ethereum-optimism/op-geth/blob/831e3fd7447f32be6a6d8e06b674479c400e7416/core/types/transaction_signing.go#L179-L184) is that the latter supports Blob transactions. However, Blob transactions are meant to be posted on L1 and are not intended for inclusion in L2 blocks. In fact, such transactions are discarded from the L2 [`txpool`](https://github.com/mantlenetworkio/op-geth/blob/4a20fa61a79e4c4cd61138e467332368d7fc8a8d/core/txpool/txpool.go#L637-L640). Despite this, since `LatestSignerForChainID` returns `cancunSigner`, it allows users to create Blob transactions for the L2 by calling [`NewKeyedTransactionWithChainID`](https://github.com/mantlenetworkio/op-geth/blob/4a20fa61a79e4c4cd61138e467332368d7fc8a8d/accounts/abi/bind/auth.go#L144-L164) and providing a Blob transaction, even though that transaction will be rejected.

To ensure uniform behavior of the three functions and to prevent attempts to submit Blob transactions to the L2 chain, consider modifying `LatestSignerForChainID` to return `londonSigner`. This change would align the behavior of all three functions and maintain the intended handling of transactions on L2.

***Update:** Acknowledged, will resolve. The Mantle team stated:*

> *Acknowledged. It will be fixed in the next version.*

---
### Example 2

**Auto Label:** Failure to validate message origin or target leads to unauthorized fund transfers, double-spending, or permanent loss of assets due to insufficient cross-domain authentication and state consistency checks.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/405 

## Found by 
Ali9955, Tendency, ZeroTrust, novaman33

## Summary

`InfernalRiftBelow.thresholdCross` verify the wrong `msg.sender`, `thresholdCross` will fail to be called, resulting in the loss of user assets.

## Vulnerability Detail

`thresholdCross` determines whether `msg.sender` is `expectedAliasedSender`:

```solidity
    address expectedAliasedSender = address(uint160(INFERNAL_RIFT_ABOVE) + uint160(0x1111000000000000000000000000000000001111));
    // Ensure the msg.sender is the aliased address of {InfernalRiftAbove}
    if (msg.sender != expectedAliasedSender) {
        revert CrossChainSenderIsNotRiftAbove();
    }
```

but in fact the function caller should be `RELAYER_ADDRESS`,
In `sudoswap`, `crossTheThreshold` check whether `msg.sender` is `RELAYER_ADDRESS`:
https://github.com/sudoswap/InfernalRift/blob/7696827b3221929b3fa563692bd4c5d73b20528e/src/InfernalRiftBelow.sol#L56


L1 across chain message through the `PORTAL.depositTransaction`, rather than `L1_CROSS_DOMAIN_MESSENGER`.

To avoid confusion, use in L1 should all `L1_CROSS_DOMAIN_MESSENGER.sendMessage` to send messages across the chain, avoid the use of low level `PORTAL. depositTransaction` function.

```solidity
 function crossTheThreshold(ThresholdCrossParams memory params) external payable {
        ......
        // Send package off to the portal
        PORTAL.depositTransaction{value: msg.value}(
            INFERNAL_RIFT_BELOW,
            0,
            params.gasLimit,
            false,
            abi.encodeCall(InfernalRiftBelow.thresholdCross, (package, params.recipient))
        );

        emit BridgeStarted(address(INFERNAL_RIFT_BELOW), package, params.recipient);
    }
```

## Impact
When transferring nft across chains,`thresholdCross` cannot be called in L2, resulting in loss of user assets.

## Code Snippet
https://github.com/sherlock-audit/2024-08-flayer/blob/0ec252cf9ef0f3470191dcf8318f6835f5ef688c/moongate/src/InfernalRiftBelow.sol#L135-L145

## Tool used

Manual Review

## Recommendation
```solidity
    // Validate caller is cross-chain
    if (msg.sender != RELAYER_ADDRESS) { //or L2_CROSS_DOMAIN_MESSENGER
        revert NotCrossDomainMessenger();
    }

    // Validate caller comes from {InfernalRiftBelow}
    if (ICrossDomainMessenger(msg.sender).xDomainMessageSender() != InfernalRiftAbove) {
        revert CrossChainSenderIsNotRiftBelow();
    }
```



## Discussion

**zhaojio**

We asked sponsor:
> InfernalRiftBelow.claimRoyalties function of the msg.sender, is RELAYER_ADDRESS? right??

Sponsor reply:
> good catch, it should be L2_CROSS_DOMAIN_MESSENGER instead

---
### Example 3

**Auto Label:** Failure to validate payload fields and signer consistency leads to acceptance of invalid or unsupported transactions, causing incorrect state transitions and denial of service.  

**Original Text Preview:**

## Description

This section details miscellaneous findings discovered by the testing team that do not have direct security implications:

1. **muxdemux.rs is unused**:  
   The file and associated code is not used anywhere in Reth and could be deleted. Consider removing the unused code if not otherwise useful (e.g. intended for library users).

2. **Unnecessary, potentially lossy casts**:  
   At `crates/storage/nippy-jar/src/lib.rs`, `DataReader::offset_size` is stored as a `u64` type and typically used by casting it to a `usize` like `self.offset_size as usize`. This can be a lossy, potentially truncating cast on some architectures where the `usize` type is smaller than a `u64` (like 32bit x86). This can be avoided by instead defining `offset_size` as a `u8` type, which is sufficient to hold the value retrieved from disk.

3. **Confusing blockchain-tree function naming**:  
   (a) The function `BlockChainTree::is_block_hash_inside_chain()` (defined at `crates/blockchain-tree/src/blockchain_tree.rs:226`) is a confusing and misleading name. From the implementation and doc comment, it is clear that the functionality is more aptly described with "inside-chain" rather than "inside chain" (as it first appears). Consider renaming to `is_block_hash_in_sidechain` or similar.  
   The same struct also has a very similarly named function `is_block_inside_chain()` that returns whether a block is present in the `BlockIndices`, and has a doc comment including "inside chain". Evaluate whether `is_block_inside_chain` should also be renamed.  
   (b) In `crates/blockchain-tree/src/block_indices.rs`, the comments for `get_canonical_block_number()` and `is_block_hash_canonical()` mention a “canonical chain,” but the implementation is only checking canonical non-finalized blocks. This is also true for `get_canonical_block_number()` defined at `crates/blockchain-tree/src/canonical_chain.rs:49`. Consider renaming to reduce confusion, and ensure doc comments are clear.  
   (c) Similarly, `crates/blockchain-tree/src/canonical_chain.rs` defines `get_canonical_block_number()` and `canonical_number()` functions, which are quite ambiguously named. On inspection, `get_canonical_block_number()` checks through non-finalized canonical blocks, and `canonical_number()` iterates over the whole canonical chain cached in the index. Consider renaming and making doc comments more clear.

4. **Validation of non-terminal difficulty blocks not implemented**:  
   Noted as a TODO, if a consensus client sends a block before total terminal difficulty then it will not be properly validated. This should be fine since the consensus client will send via `engine_newPayload` from the Engine API which sets difficulty to zero for new blocks. Only an issue during syncing if malformed blocks are received.

5. **Unused functions may be removed**:  
   The following functions are unused and may be removed:  
   - `validate_transaction_regarding_header()`  
   - `validate_all_transaction_regarding_block_and_nonces()`

6. **Chain split allows block number larger than the chain**:  
   If the function `split()` is called with a block number greater than the tip, it will consider this split valid and treat it the same as splitting the canonical head. If `block_number > chain_tip`, then it implies the block is not in this chain and the chain cannot be split. Consider returning an error for this case instead of `NoSplitPending`.

7. **reth-codecs documentation issues**:  
   (a) In `crates/storage/codecs/README.md`, the Features section is out-of-date. There is no mention of the compact encoding that is now the default (main codec).  
   (b) There is no documentation detailing or specifying the Compact encoding format. The `docs/design/codecs.md` file is empty, the crate README.md does not mention the format, and the doc comments give little detail in terms of how the standard and varuint types are encoded.

8. **reth-blockchain-tree documentation issues**:  
   (a) At `crates/blockchain-tree/src/blockchain_tree.rs:138`, the doc comment states that `is_block_known()` returns an error if “the block is already finalized.” This is not accurate, and it is more clear to state that an error is returned when the block is not part of the canonical chain but is at a height that is already finalized. (The block itself is not finalized.)

9. **Project structure documentation issues**:  
   At `docs/repo/ci.md`, line [10] contains a broken link to a fuzz GitHub workflow that does not currently exist.

## Recommendations
Ensure that the comments are understood and acknowledged, and consider implementing the suggestions above.

## Resolution
The development team have acknowledged these findings, addressing them where appropriate as follows:
1. Resolved in PR #8287.
2. Resolved in PR #8360.
3. Resolved in PR #8408.
4. Unresolved.
5. Resolved in PR #7972.
6. Resolved in PR #8285.
7. Resolved in PR #8665.
8. Resolved in PR #8408.
9. Resolved in PR #8363.

## RETH-51 Missing Payload Header Validation For Blob Fields, Withdrawals  
**Asset**: `crates/payload/validator/src/lib.rs`  
**Status**: Resolved: See Resolution  
**Rating**: Informational  

### Description
The function `ensure_well_formed_payload()` lacks checks for the fields `block.header.blob_gas_used` and `block.header.excess_blob_gas` to ensure they are `None` before the Cancun update timestamp. For a V3 execution payload, the function `try_payload_v3_to_block()` allows setting `blob_gas_used` and `excess_blob_gas` to `Some()` value. There lacks checks to ensure these values are set to `None` before Cancun. Note similarly for Shanghai, Withdrawals may be added to blocks prior to the Shanghai upgrade. Note that although the fork is passed, if blocks are sent with a timestamp before Shanghai, these should not have withdrawals. Furthermore, `cancun_fields.parent_beacon_block_root()` should only return `Some()` after the Cancun upgrade. If the value is `Some()`, it will be set in the function `try_into_block()`.

```rust
pub fn ensure_well_formed_payload(
    &self,
    payload: ExecutionPayload,
    cancun_fields: MaybeCancunPayloadFields,
) -> Result<SealedBlock, PayloadError> {
    let block_hash = payload.block_hash();

    // First parse the block
    let block = try_into_block(payload, cancun_fields.parent_beacon_block_root())?; // @audit may set `parent_beacon_block_root` before Cancun,→
    let cancun_active = self.is_cancun_active_at_timestamp(block.timestamp);
    if !cancun_active && block.has_blob_transactions() { // @audit lacks checks for block.header excess_blob_gas and blob_gas_used
        // cancun not active but blob transactions present
        return Err(PayloadError::PreCancunBlockWithBlobTransactions)
    }

    // @audit should include checks for Shanghai withdrawals

    // Ensure the hash included in the payload matches the block hash
    let sealed_block = validate_block_hash(block_hash, block)?;
    // EIP-4844 checks
    self.ensure_matching_blob_versioned_hashes(&sealed_block, &cancun_fields)?;

    Ok(sealed_block)
}
```

The severity is rated as informational severity as the endpoint is only callable through the authenticated EngineAPI and the consensus layer “should” be calling the correct version based on the incoming block timestamp.

### Recommendations
Add checks to `ensure_well_formed_payload()` to ensure that these fields are `None` before Cancun and `Some()` afterwards. Additionally, add checks for the Shanghai fork to ensure that withdrawals are not included before this timestamp and are included afterwards. Finally, ensure `cancun_fields.fields` is `None` before Cancun and `Some()` afterwards.

### Resolution
The issue was resolved in PR #7993 and alloy PR #649.

---
