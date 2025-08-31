# Cluster -1153

**Rank:** #354  
**Count:** 17  

## Label
Clock-based state validation failures leading to invalid state transitions, premature clock exhaustion, and unfair or redundant claim execution in dispute resolution.

## Cluster Information
- **Total Findings:** 17

## Examples

### Example 1

**Auto Label:** **Non-deterministic execution in consensus logic leading to state divergence and chain halts.**  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-12-seda-protocol-judging/issues/245 

## Found by 
g

### Summary

In Tally module's `EndBlock()`, all tallying Data Requests will be [processed](https://github.com/sherlock-audit/2024-12-seda-protocol/blob/main/seda-chain/x/tally/keeper/endblock.go#L40). Each Data Request can have a [Consensus Filter](https://github.com/sherlock-audit/2024-12-seda-protocol/blob/main/seda-chain/x/tally/keeper/endblock.go#L200), which will be [applied](https://github.com/sherlock-audit/2024-12-seda-protocol/blob/main/seda-chain/x/tally/keeper/filter.go#L77) to every reveal object in the Data Request. 

When the filter type is [`FilterStdDev`](https://github.com/sherlock-audit/2024-12-seda-protocol/blob/main/seda-chain/x/tally/types/filters.go#L190-L193) or [`FilterMode`](https://github.com/sherlock-audit/2024-12-seda-protocol/blob/main/seda-chain/x/tally/types/filters.go#L80-L93), the consensus filter will be applied. The filter is treated as a path expression for querying data from a JSON object. One of the supported path expressions is the wildcard expression, which gets all the elements in the JSON object, but the results have a non-deterministic order. Due to this non-deterministic order, validators will [get different](https://github.com/sherlock-audit/2024-12-seda-protocol/blob/main/seda-chain/x/tally/types/filters_util.go#L58-L61) `dataList`, `freq`, and `maxFreq`. This leads to a state divergence that will cause consensus failures, and ultimately, a chain halt.

### Root Cause

In [`filters_util.go:36-51`](https://github.com/sherlock-audit/2024-12-seda-protocol/blob/main/seda-chain/x/tally/types/filters_util.go#L36-L51), any path expression is accepted and it expects that the results
will have a deterministic ordering. Only the 0th-index of `elems` is accessed.

```golang
    obj, err := parser.Parse(revealBytes)
    if err != nil {
      errors[i] = true
      continue
    }
    expr, err := jp.ParseString(dataPath)
    if err != nil {
      errors[i] = true
      continue
    }
    // @audit the path exression is applied here to query elements from the reveal JSON object
    elems := expr.GetNodes(obj)
    if len(elems) < 1 {
      errors[i] = true
      continue
    }
    // @audit only the first element is returned as data
    data := elems[0].String()
```

Below is an example of a wildcard expression used to query a JSON object.

```pseudocode
JSON: {"a": 1, "b": 2, "c": 3}
Expression: "$.*"
Results could be: [1,2,3] or [2,1,3] or [3,1,2] etc.
```

The [results](https://github.com/sherlock-audit/2024-12-seda-protocol/blob/main/seda-chain/x/tally/keeper/filter.go#L77) of applying the filter will be in the form of `outliers` and `consensus`, which are a `[]bool` and `bool`. 

```golang
outliers, consensus := filter.ApplyFilter(reveals, res.Errors)
```

`outliers` and `consensus` can be different values for different validators. These values affect the output data results, which
will be [stored](https://github.com/sherlock-audit/2024-12-seda-protocol/blob/main/seda-chain/x/tally/keeper/endblock.go#L109-L141).

```golang
    _, tallyResults[i] = k.FilterAndTally(ctx, req, params, gasMeter)
    // @audit the Result, ExitCode, and Consensus can be different across validators because of the non-deterministic
    // results of applying the filter
    dataResults[i].Result = tallyResults[i].Result
    dataResults[i].ExitCode = tallyResults[i].ExitCode
    dataResults[i].Consensus = tallyResults[i].Consensus
    // ... snip ...
  }

  processedReqs[req.ID] = k.DistributionsFromGasMeter(ctx, req.ID, req.Height, gasMeter, params.BurnRatio)

  dataResults[i].GasUsed = gasMeter.TotalGasUsed()
  dataResults[i].Id, err = dataResults[i].TryHash()
  // ... snip ...

// Store the data results for batching.
for i := range dataResults {
  // @audit the data results are stored, but since the data will be different across validators, there will be
  // state divergence.
  err := k.batchingKeeper.SetDataResultForBatching(ctx, dataResults[i])
```

### Internal Pre-conditions
None


### External Pre-conditions
None


### Attack Path

A malicious user can abuse this.
1. A malicious user can post multiple valid Data Requests with a wildcard expression `$.*` as consensus filter.
2. Once the valid Data Requests are in "Tallying Status", the Tally module will process them and store their corresponding
Data Results for batching. Every validator here will store different values for their data results.
3. There will be a Chain Halt because there will be no consensus on the state root across validators.


### Impact

Chain halt due to state divergence across validators.

### PoC
None


### Mitigation
Consider always sorting the [result](https://github.com/sherlock-audit/2024-12-seda-protocol/blob/main/seda-chain/x/tally/types/filters_util.go#L46) of `expr.GetNodes(obj)` before getting the first element as the result.

## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/sedaprotocol/seda-chain/pull/525

---
### Example 2

**Auto Label:** **Non-deterministic execution in consensus logic leading to state divergence and chain halts.**  

**Original Text Preview:**

##### Description

There is a possible non-determinism issue in `FinalizeBlock`. That function must be deterministic per the CometBFT specs. However, there is a call path `postFinalize -> isNextProposer -> Validators -> cmtAPI` that can incurr in a non-deterministic scenario:

  

<https://github.com/piplabs/story/blob/e6d2d51550c3eff3f561f8d1b860888ea2bf8060/client/app/abci.go#L114>

```
// FinalizeBlock calls BeginBlock -> DeliverTx (for all txs) -> EndBlock.
func (l abciWrapper) FinalizeBlock(ctx context.Context, req *abci.RequestFinalizeBlock) (*abci.ResponseFinalizeBlock, error) {

	...

	if err := l.postFinalize(sdkCtx); err != nil {
		log.Error(ctx, "PostFinalize callback failed [BUG]", err, "height", req.Height)
		return resp, err
	}

        ...
```

  

<https://github.com/piplabs/story/blob/e6d2d51550c3eff3f561f8d1b860888ea2bf8060/client/x/evmengine/keeper/abci.go#L189>

```
// PostFinalize is called by our custom ABCI wrapper after a block is finalized.
// It starts an optimistic build if enabled and if we are the next proposer.
//
// This custom ABCI callback is used since we need to trigger optimistic builds
// immediately after FinalizeBlock with the latest app hash
// which isn't available from cosmosSDK otherwise.
func (k *Keeper) PostFinalize(ctx sdk.Context) error {

	...

	// Maybe start building the next block if we are the next proposer.
	isNext, err := k.isNextProposer(ctx, proposer, height)
	if err != nil {
		return errors.Wrap(err, "next proposer")
	}

        ... 
```

  

<https://github.com/piplabs/story/blob/e6d2d51550c3eff3f561f8d1b860888ea2bf8060/client/x/evmengine/keeper/keeper.go#L191>

```
// isNextProposer returns true if the local node is the proposer
// for the next block. It also returns the next block height.
//
// Note that the validator set can change, so this is an optimistic check.
func (k *Keeper) isNextProposer(ctx context.Context, currentProposer []byte, currentHeight int64) (bool, error) {

	...

	valset, ok, err := k.cmtAPI.Validators(ctx, currentHeight)
	if err != nil {
		return false, err
	}

        ...
```

  

The reason why it can lead to non-determinism is because the call to the CometBFT API is done through an RPC, which is a network connection, which by definition is non-deterministic between nodes (as some can revert due to timeout, network errors or go on flawlessly). That makes it possible that, under the same input, some nodes would go on whilst others would crash, as the errors are not handled in the `FinalizeBlock` context but rather bubbled up to the caller, leading to a chain split and the consensus would be broken.

##### BVSS

[AO:A/AC:L/AX:M/C:N/I:N/A:H/D:N/Y:N/R:N/S:U (5.0)](/bvss?q=AO:A/AC:L/AX:M/C:N/I:N/A:H/D:N/Y:N/R:N/S:U)

##### Recommendation

Ignore any error returned from `PostFinalize` in `FinalizeBlock`.

##### Remediation

**SOLVED:** The **Story team** fixed this issue by skipping optimistic builds if any non-determinism error was encountered.

##### Remediation Hash

<https://github.com/piplabs/story/commit/596907db4987212ab59c8d7f5019b86157134dfa>

---
### Example 3

**Auto Label:** **Non-deterministic execution in consensus logic leading to state divergence and chain halts.**  

**Original Text Preview:**

## Severity: Low Risk

## Context
(No context files were provided by the reviewers)

## Description
```go
// VerifySignature verifies a signature against a message and a public key.
func (f BLSSigner) VerifySignature(
    pubKey crypto.BLSPubkey,
    msg []byte,
    signature crypto.BLSSignature,
) error {
    if ok := bls12381.PubKey(pubKey[:]).VerifySignature(msg, signature[:]); !ok {
        return ErrInvalidSignature
    }
    return nil
}
```

The `beacon-kit` uses BLS keys extensively for signing messages in CometBFT validator set and verifying messages signed by other validators. The CometBFT version used in the signer package is `github.com/cometbft/cometbft v1.0.0-rc1.0.20240806094948-2c4293ef36c4`. This version accepts public keys which are outside of the permitted subgroup due to loose validation checks.

Due to this, it can force computations and operations on a larger subgroup which are expensive and can consume more resources on the node. However, computations are still mathematically valid and deterministic; there is still unknown risk due to unexpected cryptographic properties.

## Recommendation
The BLS module was fixed in CometBFT in PR 4104. Upgrade CometBFT version to a more recent commit in all packages.

- **Berachain:** Fixed in PR 2221.
- **Spearbit:** Fix verified.

---
