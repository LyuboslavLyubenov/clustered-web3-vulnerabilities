# Cluster -1016

**Rank:** #338  
**Count:** 19  

## Label
Failure to validate or update state upon blacklisting leads to incorrect allocations, denial of service, and loss of yield optimization due to inconsistent or blocked protocol state progression.

## Cluster Information
- **Total Findings:** 19

## Examples

### Example 1

**Auto Label:** Race conditions and logic errors in state transitions, leading to incorrect state validation, invalid challenge processing, and compromised protocol functionality.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-12-seda-protocol-judging/issues/239 

## Found by 
g, zxriptor

### Summary

When a proving scheme is ready for activation, all validators without registered keys will be jailed. However, there is no check
that a validator is currently jailed before jailing, which raises an error and causes the `EndBlock()` to return early without
activating the proving scheme. 

### Root Cause

- In Pubkey module's `EndBlock()`, a validator without keys will get [jailed](https://github.com/sherlock-audit/2024-12-seda-protocol/blob/main/seda-chain/x/pubkey/keeper/endblock.go#L135-L144) without first checking if it is already jailed.
- `slashingKeeper.Jail()` eventually calls [`jailValidator()`](https://github.com/cosmos/cosmos-sdk/blob/v0.50.11/x/staking/keeper/val_state_change.go#L309-L311), which returns an error when a validator is already jailed.
```golang
if validator.Jailed {
  return types.ErrValidatorJailed.Wrapf("cannot jail already jailed validator, validator: %v", validator)
}
```

### Internal Pre-conditions
None


### External Pre-conditions
None


### Attack Path
1. A validator has no registered keys. It is [optional](https://github.com/sherlock-audit/2024-12-seda-protocol/blob/main/seda-chain/x/staking/keeper/msg_server.go#L65-L76) to register keys while the scheme is not activated.
2. This validator with no keys gets permanently Jailed for double-signing.
3. The Pubkey module's `EndBlock()` always [returns early](https://github.com/sherlock-audit/2024-12-seda-protocol/blob/main/seda-chain/x/pubkey/keeper/endblock.go#L40-L43), because `JailValidators()` always fails.


### Impact

The Proving Scheme will not be activated at the configured activation height and will remain inactive while the validator is jailed
and has no registered key. A validator can be jailed permanently, leading to the Proving Scheme never getting activated.

A validator can prevent [batches](https://github.com/sherlock-audit/2024-12-seda-protocol/blob/main/seda-chain/x/batching/keeper/endblock.go#L36-L42) from ever getting produced because the SEDAKeyIndexSecp256k1 proving scheme never gets activated.


### PoC
None


### Mitigation
Consider checking first if the Validator is Jailed before jailing it.

## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/sedaprotocol/seda-chain/pull/522

---
### Example 2

**Auto Label:** Failure to validate or update state upon blacklisting leads to incorrect allocations, denial of service, and loss of yield optimization due to inconsistent or blocked protocol state progression.  

**Original Text Preview:**

## Severity: Medium Risk

## Context
- AnchorStateRegistry.sol#L109-L111
- AnchorStateRegistry.sol#L266-L273
- FaultDisputeGame.sol#L299-L300
- OPContractsManager.sol#L541-L544

## Description
`AnchorStateRegistry.getAnchorRoot()` checks if the current `anchorGame` is blacklisted and reverts if so:

```solidity
if (isGameBlacklisted(anchorGame)) {
    revert AnchorStateRegistry_AnchorGameBlacklisted();
}
```

While this is meant to prevent invalid anchor roots from being used, `getAnchorRoot()` reverting will break several important functions in the protocol.

`anchorGame` could be blacklisted if the following sequence occurs:
1. Dispute game incorrectly resolves to `DEFENDER_WINS`.
2. `DISPUTE_GAME_FINALITY_DELAY_SECONDS` passes.
3. Guardian calls `blacklistDisputeGame()` to blacklist the dispute game.
4. Attacker front-runs guardian, calling `setAnchorState()` to set the dispute game as `anchorGame`.

When this occurs, due to `getAnchorRoot()` always reverting when called:
1. New dispute games cannot be created as `FaultDisputeGame.initialize()` reverts.
2. `anchorGame` cannot be updated as `AnchorStateRegistry.setAnchorState()` reverts.
3. The chain cannot be upgraded as `OPContractsManager.upgrade()` reverts.

(1) is particularly severe as there is no way for Optimism's proposer (or anyone else) to submit root claims to L1 until the situation is manually handled. This violates L2Beat's Stage 1 requirements.

Additionally, since `setAnchorState()` cannot be called as outlined in (2), `anchorGame` is permanently set to the blacklisted dispute game. As such, there is no way for such a situation to be resolved without an upgrade to `AnchorStateRegistry`.

## Recommendation
Remove the blacklist check from `getAnchorRoot()`. If there is a need to ensure the anchor root is not invalid, it should be checked before calling `getAnchorRoot()`. For example, `OPContractsManager.upgrade()` could check if `anchorGame` is blacklisted before calling `anchors()`.

Note that the only way to prevent a blacklisted `anchorGame` from occurring is to ensure the guardian will always blacklist an incorrectly resolved dispute game before `DISPUTE_GAME_FINALITY_DELAY_SECONDS` has passed (currently 3.5 days after the dispute game is resolved). This is fully reliant on the guardian and nothing in the code prevents `anchorGame` from being blacklisted.

## Optimism
Fixed in PR 14232 by removing the blacklist check. This should be safe under the assumption that invalid dispute games are invalidated before they are finalized. Additionally, we will begin to formalize the L2Beat Stage 1 definition as an invariant that any future design must maintain.

## Spearbit
Verified, the blacklist check in `getAnchorRoot()` was removed.

---
### Example 3

**Auto Label:** Race conditions and logic errors in state transitions, leading to incorrect state validation, invalid challenge processing, and compromised protocol functionality.  

**Original Text Preview:**

## Summary

The `getBestResponse` function in `LLMOracleCoordinator` lacks a tiebreak mechanism when multiple responses have the same highest validation score.

This can lead to inconsistent results and potential manipulation of which response is selected as "best".

## Vulnerability Details

Current implementation simply takes the first response with the highest score:

```Solidity
function getBestResponse(uint256 taskId) external view returns (TaskResponse memory) {
    TaskResponse[] storage taskResponses = responses[taskId];

    // ensure that task is completed
    if (requests[taskId].status != LLMOracleTask.TaskStatus.Completed) {
        revert InvalidTaskStatus(taskId, requests[taskId].status, LLMOracleTask.TaskStatus.Completed);
    }

    // pick the result with the highest validation score
    TaskResponse storage result = taskResponses[0];
    uint256 highestScore = result.score;
    for (uint256 i = 1; i < taskResponses.length; i++) {
        if (taskResponses[i].score > highestScore) {  // Note: only strictly greater than
            highestScore = taskResponses[i].score;
            result = taskResponses[i];
        }
    }

    return result;
}
```

Issues:

No tiebreaker for equal scores

First response has advantage in ties

Order-dependent results

## Impact

Early responders have advantage in ties

Inconsistent selection among equally-scored responses

## Tools Used

Manual Review

## Recommendations

Implement deterministic tiebreak using multiple factors:

```Solidity
function getBestResponse(uint256 taskId) external view returns (TaskResponse memory) {
    TaskResponse[] storage taskResponses = responses[taskId];
    require(requests[taskId].status == LLMOracleTask.TaskStatus.Completed, "Task not completed");

    TaskResponse storage bestResponse = taskResponses[0];
    uint256 bestScore = bestResponse.score;
    bytes32 bestHash = keccak256(abi.encodePacked(
        bestResponse.output,
        bestResponse.responder,
        bestResponse.nonce
    ));

    for (uint256 i = 1; i < taskResponses.length; i++) {
        TaskResponse storage currentResponse = taskResponses[i];
        uint256 currentScore = currentResponse.score;
        
        // If strictly better score, always choose it
        if (currentScore > bestScore) {
            bestResponse = currentResponse;
            bestScore = currentScore;
            bestHash = keccak256(abi.encodePacked(
                currentResponse.output,
                currentResponse.responder,
                currentResponse.nonce
            ));
        }
        // If tied score, use deterministic tiebreak
        else if (currentScore == bestScore) {
            bytes32 currentHash = keccak256(abi.encodePacked(
                currentResponse.output,
                currentResponse.responder,
                currentResponse.nonce
            ));
            // Use hash comparison as tiebreaker
            if (currentHash < bestHash) {
                bestResponse = currentResponse;
                bestHash = currentHash;
            }
        }
    }

    return bestResponse;
}
```

---
