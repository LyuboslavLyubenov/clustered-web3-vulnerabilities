# Cluster -1223

**Rank:** #347  
**Count:** 18  

## Label
Missing input validation in configuration updates leads to inconsistent state, arithmetic overflows, or invalid state transitions, enabling denial-of-service, asset drain, or consensus-breaking behavior.

## Cluster Information
- **Total Findings:** 18

## Examples

### Example 1

**Auto Label:** Missing or invalid validation of critical consensus parameters leads to insecure defaults, premature execution, slot mismatches, and denial-of-service, undermining finality and chain integrity.  

**Original Text Preview:**

The function `Beacon.sol#_buildCmd()` sets the value of the confirmation to zero.

```solidity
EVMCallComputeV1 memory computeSettings = EVMCallComputeV1(
            _MAP_AND_REDUCE,
            THIS_CONTRACT_EID,
            false,
            uint64(block.timestamp),
            0,//confirmations
            address(this)
        );
```

It is the number of confirmations required to wait for the timestamp finality on the target chain.
This can be resolved by setting a reasonable number.

---
### Example 2

**Auto Label:** Missing or invalid validation of critical consensus parameters leads to insecure defaults, premature execution, slot mismatches, and denial-of-service, undermining finality and chain integrity.  

**Original Text Preview:**

## Severity: Low Risk

## Context
`beacon/blockchain/receive.go#L140`

## Description
From my understanding, the `s.optimisticPayloadBuilds` in Beacon-Kit determines if we should send an `engine_forkchoiceUpdatedV2` containing `PayloadAttributes` to have the EL client start building a new `ExecutionPayload` for the next slot. This is useful for a validator that is going to require a built `ExecutionPayload` when it becomes the proposer in the next round. That being said, every single validator in beacon-kit is optimistically building a payload every single slot, even if they are not going to be the next proposer. This is a ton of computation that doesn't need to happen.

## Recommendation
In cometBFT, the next validator that is going to be the proposer is determined by a deterministic process that I believe can be calculated ahead of time by just knowing the current validator set. The `shouldBuildOptimisticPayloads()` function should determine if the local validator is going to be the proposer in the next slot. Otherwise, it should not build the optimistic payload.

## Berachain
Acknowledged. A fix for this is being considered.

## Spearbit
Acknowledged.

---
### Example 3

**Auto Label:** Missing or invalid validation of critical consensus parameters leads to insecure defaults, premature execution, slot mismatches, and denial-of-service, undermining finality and chain integrity.  

**Original Text Preview:**

## Low Risk Issue Report

**Severity:** Low Risk  
**Context:** `beacon/blockchain/execution_engine.go#L47-L51`  

## Description
After finalizing a block, Beacon-Kit has the following check when sending the new `engine_fork-choiceUpdatedV3` to the EL client:

```go
if !s.shouldBuildOptimisticPayloads() && s.localBuilder.Enabled() {
    s.sendNextFCUWithAttributes(ctx, st, blk, lph)
} else {
    s.sendNextFCUWithoutAttributes(ctx, blk, lph)
}
```

If the node is not an optimistic builder, `s.shouldBuildOptimisticPayloads()` returns false. Then `sendNextFCUWithAttributes()` is called, which sends an `engine_forkchoiceUpdatedV3` containing `PayloadAttributes`. This triggers a build on the EL client for the next slot. 

This means that every single node, even if it has optimistic builds disabled, is requesting optimistic builds on every finalized block.

## Recommendation
Remove the `sendNextFCUWithAttributes()` function entirely. In `sendPostBlockFCU()`, Beacon-Kit should only send an `engine_forkchoiceUpdatedV3` request without attributes. This is because an optimistic client has already requested an optimistic build in `handleOptimisticPayloadBuild()`, and the non-optimistic client does not need to request an optimistic build in the first place.

This function should only be using `engine_forkchoiceUpdatedV3` to notify the EL client of the new head.

**Berachain:** Fixed in PR 2240.  
**Spearbit:** Fix verified.

---
