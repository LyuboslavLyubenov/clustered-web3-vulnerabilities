# Cluster -1048

**Rank:** #375  
**Count:** 14  

## Label
Unchecked loop increments cause out-of-bounds access or infinite loops, leading to state corruption and potential revenue loss due to unvalidated iteration bounds.

## Cluster Information
- **Total Findings:** 14

## Examples

### Example 1

**Auto Label:** Infinite loops and incorrect loop termination due to flawed control flow, improper bounds checking, and race conditions in iterative or recursive execution.  

**Original Text Preview:**

## car.BeaconBlockHeader.StateRoot

**Severity:** Medium Risk  
**Context:** `beacon/validator/block_builder.go#L108-L124`  

## Description
While building the `BeaconBlock` and `BlobSidecars` for proposal, there are the following two goroutines:

```go
g.Go(func() error {
    sidecars, err = s.blobFactory.BuildSidecars(
        blk, envelope.GetBlobsBundle(),
    )
    return err
})

// Compute the state root for the block.
g.Go(func() error {
    return s.computeAndSetStateRoot(
        ctx,
        slotData.GetProposerAddress(),
        slotData.GetConsensusTime(),
        st,
        blk,
    )
})
```

The first goroutine calls `BuildSidecars`, which calls `blk.GetHeader()`. This reads the `StateRoot` of the `BeaconBlock` and places it into the `BlobSidecar`. The second goroutine calculates and sets the `StateRoot` in the `BeaconBlock`. They both happen at the same time—this could result in a mismatched state root between the `BeaconBlock.StateRoot` and each `BlobSidecar.BeaconBlockHeader.StateRoot`.

## Recommendation
There are two options:

1. The simplest option would be to serialize these two calls. First, calculate the state root and then build the sidecars.
2. If you want to keep the benefits of parallelization, you can set the `BlobSidecar.BeaconBlockHeader.StateRoot` to a placeholder value during the parallel goroutines. After both goroutines are complete, then each `BlobSidecar.BeaconBlockHeader.StateRoot` value can be updated to the new `BeaconBlock.StateRoot`.

**Berachain:** Fixed in commit `54047740`.  
**Spearbit:** Fix verified.

---
### Example 2

**Auto Label:** Unvalidated loops and missing boundary checks enable infinite iterations or unintended state changes, leading to reentrancy or incorrect execution.  

**Original Text Preview:**

##### Description
There is an unnecessary action in case `k == bridgeAdapterConfigs.length - 1` https://github.com/lidofinance/aave-delivery-infrastructure/blob/41c81975c2ce5b430b283e6f4aab922c3bde1555/src/contracts/CrossChainForwarder.sol#L407

##### Recommendation
We recommend adding a check that `k != bridgeAdapterConfigs.length - 1` and only in this case update the storage.

---
### Example 3

**Auto Label:** Infinite loops and incorrect loop termination due to flawed control flow, improper bounds checking, and race conditions in iterative or recursive execution.  

**Original Text Preview:**

In the `getDx()` function of the `CurvePoolUtil` library, there is a potential off-by-one error due to how the loop termination condition is checked. The loop is intended to run a maximum number of iterations defined by `MAX_ITERATIONS_BINSEARCH`. However, the condition `if (loops > MAX_ITERATIONS_BINSEARCH)` allows the loop to execute one more iteration than intended.

Consider adjusting the condition to `if (loops >= MAX_ITERATIONS_BINSEARCH)`. This adjustment will ensure that the loop runs for exactly `MAX_ITERATIONS_BINSEARCH` iterations at most, aligning with the intended logic.

---
