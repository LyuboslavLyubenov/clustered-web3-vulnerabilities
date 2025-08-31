# Cluster -1249

**Rank:** #262  
**Count:** 33  

## Label
Inconsistent payload validation and cryptographic mismatch due to flawed field inclusion, incorrect hashing, and improper message parsing, leading to unauthorized operations and potential fund lockups.

## Cluster Information
- **Total Findings:** 33

## Examples

### Example 1

**Auto Label:** Inconsistent payload validation and cryptographic mismatch due to flawed field inclusion, incorrect hashing, and improper message parsing, leading to unauthorized operations and potential fund lockups.  

**Original Text Preview:**

## SwiftSource Implementation Issues

In the current implementation of `SwiftSource::unlockCompressedBatch`, the action check incorrectly verifies for `BATCH_UNLOCK` instead of `COMPRESSED_UNLOCK`. Furthermore, the current encoded payload format parser in `SwiftSource` does not follow the `SwiftDest` payload format. The `msgHash` is read starting at byte 1. However, the `SwiftDest` payload format specifies that the `msgHash` starts at byte 3. Thus, if the `msgHash` is extracted from the wrong offset, the computed hash may not match the expected value.

Also, the current implementation passes an offset of 33 to `processUnlocks`, which is meant to skip 1 byte for action and 32 bytes for the `msgHash`. However, the `encodedPayload` only contains the packed `unlockMsgs` and their length, not the full Wormhole payload. This implies the action and `msgHash` are not present in `encodedPayload`, and the offset should not skip 33 bytes.

## Remediation

1. Update the action check such that it is verifying for `COMPRESSED_UNLOCK`.
2. Adjust the `msgHash` offset to 3.
3. Update the call to `processUnlocks` to start at 0.

## Patch

Resolved in commit `9e5981b`.

---
### Example 2

**Auto Label:** Inconsistent payload validation and cryptographic mismatch due to flawed field inclusion, incorrect hashing, and improper message parsing, leading to unauthorized operations and potential fund lockups.  

**Original Text Preview:**

## Issues in postBatch Function within SwiftDest

There are two significant issues in the `postBatch` function within `SwiftDest`. 

## Issue 1: Incorrect Length Check

Firstly, the length check comparing the length of `unlockMsg` against the `UNLOCK_MSG_SIZE` constant is incorrect, as it reverts if they are equal. This implies that a valid unlock message will have a length different from `UNLOCK_MSG_SIZE`, which is counterintuitive since a valid message should match a predefined size. As a result, valid unlock messages will be rejected.

```solidity
// src/swift/SwiftDest.sol
function postBatch(bytes32[] memory orderHashes, bool compressed) public payable returns (uint64, sequence) {
    bytes memory encoded;
    for(uint i=0; i<orderHashes.length; i++) {
        bytes memory unlockMsg = unlockMsgs[orderHashes[i]];
        if (unlockMsg.length == UNLOCK_MSG_SIZE) {
            revert OrderNotExists(orderHashes[i]);
        }
        encoded = abi.encodePacked(encoded);
        delete unlockMsgs[orderHashes[i]];
    }
    [...]
}
```

## Issue 2: Incorrect Encoding Logic

Secondly, the encoding logic is incorrect, as currently the logic packs the `encoded` variable with itself but does not include the `unlockMsg`. Thus, the encoded payload lacks the necessary order details, rendering the batch message incomplete and creating inconsistencies in cross-chain communication.

## Remediation

- Modify the length check such that it utilizes `!=` instead of `==`.
- Update the encoding logic to include the `unlockMsg` when packing the data.

## Patch

Resolved in ab46b49.

---
### Example 3

**Auto Label:** Inconsistent payload validation and cryptographic mismatch due to flawed field inclusion, incorrect hashing, and improper message parsing, leading to unauthorized operations and potential fund lockups.  

**Original Text Preview:**

## Vulnerability Overview

The vulnerability in the `VerifyCompactUnlock` instruction arises from an inconsistency in how the hash for verifying the `compact_unlock.items` data is calculated compared to the hash calculated in the `PostUnlock` instruction. 

In the `VerifyCompactUnlock` instruction, the hash utilized for verifying the `compact_unlock.items` data does not include the items count. Conversely, in the `PostUnlock` instruction, the hash calculation explicitly includes the items count along with the data.

## Code Snippet

> _programs/swift/src/vaa/unlock_batch_compact.rs rust_

```rust
impl<'a> BatchUnlockCompactMessage<'a> {
    pub fn items_count(&self) -> u16 {
        u16::from_be_bytes(self.0[1..3].try_into().unwrap())
    }
    
    pub fn hash(&self) -> [u8; 32] {
        self.0[3..35].try_into().unwrap()
    }
    
    pub fn parse(payload: &'a [u8]) -> Result<Self> {
        if payload.len() != BATCH_UNLOCK_COMPACT_PAYLOAD_LENGTH as usize {
            msg!("payload length is wrong");
            return err!(SwiftError::InvalidUnlockBatchCompactVAA);
        }
        if payload[0] != SwiftVaaAction::UnlockBatchCompact as u8 {
            msg!("payload action is not Unlock");
            return err!(SwiftError::InvalidUnlockBatchCompactVAA);
        }
        Ok(Self(payload))
    }
    [...]
}
```

Another deviation from the `PostUnlock` implementation occurs in `BatchUnlockCompactMessage`, which expects the payload to include the `items_count` field, making the payload 35 bytes. In contrast, `PostUnlock` generates a payload of only 33 bytes, omitting the `items_count` field.

## Remediation

Update the `VerifyCompactUnlock` instruction to include the items count when calculating the hash for `compact_unlock.items`, ensuring consistency with `PostUnlock`. Also, ensure consistency between `BatchUnlockCompactMessage` and `PostUnlock`. Utilize the predefined size constants for length calculations to ensure accuracy and consistency.

## Patch

Resolved in `b0729c8`.

---
