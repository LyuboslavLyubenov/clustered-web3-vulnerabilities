# Cluster -1040

**Rank:** #323  
**Count:** 21  

## Label
Race conditions caused by unsynchronized access to shared state, leading to panics, data corruption, and incorrect message delivery due to concurrent operations without proper locking or isolation.

## Cluster Information
- **Total Findings:** 21

## Examples

### Example 1

**Auto Label:** Race conditions caused by unsynchronized access to shared state, leading to panics, data corruption, and incorrect message delivery due to concurrent operations without proper locking or isolation.  

**Original Text Preview:**

## Severity: Low Risk

## Context
`mod/async/pkg/broker/broker.go#L127-L133`

## Description
The `broker.Broker` struct includes the following map: `subscriptions map[chan T]struct{}`. This map acts as a list of all of the subscribed channels. You can add and remove channels from this map by using the `Subscribe` and `Unsubscribe` functions. However, reads and writes to this subscriptions map are not protected by any concurrency locking mechanism. During any given broadcast, a broker could `Unsubscribe`, resulting in a channel closing at the same time that it is being read from or written to. This results in a race condition in which a closed channel is written to, inducing a panic.

Here is a proof of concept that results in panic: **send on closed channel**:

```go
package main

import (
    "context"
    "github.com/berachain/beacon-kit/mod/async/pkg/broker"
    "github.com/berachain/beacon-kit/mod/async/pkg/dispatcher"
    "github.com/berachain/beacon-kit/mod/primitives/pkg/async"
)

type event struct {
    ctx  context.Context
    id   async.EventID
    data uint64
}

func (e event) ID() async.EventID {
    return e.id
}

func (e event) Context() context.Context {
    return e.ctx
}

func printer(ctx context.Context, ch chan event) {
    for {
        select {
        case <-ctx.Done():
            return
        case <-ch:
        }
    }
}

func main() {
    eventIDStr := "testevent"
    eventID := async.EventID(eventIDStr)
    ctx, cancel := context.WithCancel(context.Background())
    
    // Create the broker
    testeventBroker := broker.New[event](eventIDStr)
    
    // Create the dispatcher with nil logger and register broker
    dispatch, err := dispatcher.New(nil)
    if err != nil {
        panic("could not create dispatcher")
    }
    
    if err := dispatch.RegisterBrokers(testeventBroker); err != nil {
        panic("could not register broker")
    }
    
    if err := dispatch.Start(ctx); err != nil {
        panic("could not start dispatcher")
    }
    
    // Loop until we hit the race condition panic
    for i := 1; i < 1000; i++ {
        // Create channel and start reading from it
        ch := make(chan event)
        go printer(ctx, ch)
        
        // Subscribe the new channel
        if err := dispatch.Subscribe(eventID, ch); err != nil {
            panic("could not subscribe to event")
        }
        
        // Publish a new event
        if err := dispatch.Publish(event{
            ctx:  ctx,
            id:   eventID,
            data: uint64(i),
        }); err != nil {
            panic("could not publish event")
        }
        
        // Unsubscribe from the channel
        if err := dispatch.Unsubscribe(eventID, ch); err != nil {
            panic("could not unsubscribe from event")
        }
    }
    
    cancel()
}
```

## Recommendation
Add a `sync.RWMutex` to the type `Broker[T async.BaseEvent]` struct that protects accesses to `subscriptions`.

## Berachain
Fixed in PR 2225.

## Spearbit
Fix verified.

---
### Example 2

**Auto Label:** Race conditions in concurrent access to shared state, leading to inconsistent data, incorrect validation, or panics due to uncoordinated reads and writes across threads or transactions.  

**Original Text Preview:**

## Security Advisory

## Severity
**Medium Risk**

## Context
`execution/pkg/client/ethclient/rpc/client.go#L50`

## Description
The `rpc.Client.header` field is an `http.Header` from the Go library. This is a wrapper over a map, and it is not concurrency-safe. 

In the current usage, there is a race condition in the following spots:
- `rpc.Start()` is an infinite loop that will update the header with the `rpc.header.Set("Authorization", "Bearer " + token)`.
- `rpc.CallRaw()` will read the `rpc.header` value in a separate goroutine.

This could result in undefined behavior that could cause invalid data or panic.

## Recommendation
Protect access to the `header` http.Header field at `execution/pkg/client/ethclient/rpc/client.go#L50` with a `sync.RWMutex`.

## Additional Information
- **Berachain**: Fixed in commit `bea7c27c`.
- **Spearbit**: Fix verified.

---
### Example 3

**Auto Label:** Race conditions in concurrent access to shared state, leading to inconsistent data, incorrect validation, or panics due to uncoordinated reads and writes across threads or transactions.  

**Original Text Preview:**

## Medium Risk Severity Report

**Context:** `beacon/blockchain/process.go#L74-L78`

## Description
During the `FinalizeBlock()` call from CometBFT, Beacon-Kit will begin processing on two different event handlers at `abci.go#L345-L356`, namely `async.FinalBeaconBlockReceived` and `async.FinalSidecarsReceived`. These calls handle `handleBeaconBlockFinalization()` and `handleFinalSidecarsReceived()` respectively. These functions run in parallel with each other, which presents the following race condition:

- In `handleFinalSidecarsReceived()`, the sidecars will be written to the `AvailabilityStore` at `da/pkg/blob/processor.go#L129`.
- In `handleBeaconBlockFinalization()`, the sidecars are checked to see if they have been included in the `AvailabilityStore` by calling `IsDataAvailable()` at `beacon/blockchain/process.go#L74-L78`.

The `AvailabilityStore` is being written to and read at the same time. Thus, it can be non-deterministic whether or not the `IsDataAvailable()` check will pass during `FinalizeBlock()`. If the data is not available when the check happens, then the validator will return an error for `FinalizeBlock()`.

**Note:** This `IsDataAvailable()` check is absolutely necessary due to the fact that the block proposal is the only source of distributing `BlobSidecars`. If this check did not exist, a malicious proposer could purposefully not include a `BlobSidecar` that correlates with a `KZGCommitment` in the `BeaconBlock`. This would result in the chain continuing on without ever having the blob data made available. In my opinion, this check should happen in `ProcessProposal()`, so that an invalid proposal may be punished properly in the future.

## Recommendation
The check in `beacon/blockchain/process.go#L74-L78` should await the return of `handleFinalSidecarsReceived()`. To allow for some partial parallel processing (processing the sidecars at the same time as processing the state transition), this check could await a dispatcher event for a new event like `async.BlobDataAvailable`.

## Berachain
Fixed in commit `711ce6fd`.

## Spearbit
Fix verified.

---
