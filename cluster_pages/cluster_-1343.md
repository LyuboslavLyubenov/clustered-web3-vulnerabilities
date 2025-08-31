# Cluster -1343

**Rank:** #155  
**Count:** 89  

## Label
Failure to validate or check contract state before execution leads to incorrect state updates, unauthorized operations, or silent failures, risking asset loss, inconsistent accounting, or exploitation during invalid or transitional states.

## Cluster Information
- **Total Findings:** 89

## Examples

### Example 1

**Auto Label:** Missing validation of caller or transaction legitimacy leads to unauthorized access, spoofing, or state manipulation, enabling attacks like reentrancy, denial-of-service, or cascading failures.  

**Original Text Preview:**

**Impact**

The function `updateSystemSnapshots_excludeCollRemainder` doesn't have access control meaning that stakes can be manipulated

I found this by running invariant tests, with the simple check that no call should ever succeed


```solidity
 => [call] CryticTester.troveManager_updateSystemSnapshots_excludeCollRemainder((address,uint256)[])([]) (addr=0xA647ff3c36cFab592509E13860ab8c4F28781a66, value=0, sender=0x0000000000000000000000000000000000030000)
         => [call] TroveManager.updateSystemSnapshots_excludeCollRemainder((address,uint256)[])([]) (addr=0x48E4F3f3daE11341ff2eDF60Af6857Ae08C871C5, value=0, sender=0xA647ff3c36cFab592509E13860ab8c4F28781a66)
                 => [event] SystemSnapshotsUpdated([], [])
                 => [return ()]
         => [event] Log("should never be possible")
         => [panic: assertion failed]
```

**Mitigation**

Add a check to ensure that the caller is LiquidationOperations

---
### Example 2

**Auto Label:** Failure to validate existence or state before updating mappings leads to unauthorized access, infinite loops, or corrupted state, enabling unlimited claims, fund draining, or reentrancy.  

**Original Text Preview:**

## Security Assessment

## Severity
**Low Risk**

## Context
**File:** `OptimismPortalInterop.sol`  
**Lines:** 105-108

## Description
The `OptimismPortalInterop` contract validates that withdrawal transactions cannot target the `SharedLockbox` via `_validateWithdrawal()`. However, this check happens after the withdrawal has already been initiated on L2. A more efficient approach would be to prevent these invalid withdrawals at initiation time in the `L2ToL1MessagePasser`.

## Recommendation
Add target validation to the `initiateWithdrawal()` function in `L2ToL1MessagePasser`:

```solidity
function initiateWithdrawal(address _target, uint256 _gasLimit, bytes memory _data) public payable {
    // Check that target is not the SharedLockbox
    // Note: The exact SharedLockbox address would need to be configured per chain
    if (_target == SHARED_LOCKBOX_ADDRESS) revert InvalidTarget();
    // Also prevent targeting the OptimismPortal itself
    if (_target == OPTIMISM_PORTAL_ADDRESS) revert InvalidTarget();

    bytes32 withdrawalHash = Hashing.hashWithdrawal(
        Types.WithdrawalTransaction({
            nonce: messageNonce(),
            sender: msg.sender,
            target: _target,
            value: msg.value,
            gasLimit: _gasLimit,
            data: _data
        })
    );

    sentMessages[withdrawalHash] = true;
    emit MessagePassed(messageNonce(), msg.sender, _target, msg.value, _gasLimit, _data, withdrawalHash);
    
    unchecked {
        ++msgNonce;
    }
}
```

Note that implementing this on L2 requires some way to determine the correct target addresses since they will be specific to each chain. This could be handled through configuration or predeploys.

## OP Labs
**Acknowledged:** We don’t have the portal and lockbox addresses on the L2 side, so for now we will prevent them on the portal bad target check.

## Cantina Managed
**Acknowledged.**

---
### Example 3

**Auto Label:** Failure to validate or check for existing entries leads to state overwriting, enabling unauthorized re-deployment, loss of data, or denial of service through unintended state mutations.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-03-crestal-network-judging/issues/205 

## Found by 
0x73696d616f

### Summary

`createCommonProjectIDAndDeploymentRequest()` is called by `createAgent()`, in which the user pays fees to create an agent. The [index](https://github.com/sherlock-audit/2025-03-crestal-network/blob/main/crestal-omni-contracts/src/BlueprintCore.sol#L373) is supposed to protect the user from overwritting a requestId with the same requestId but different serverURL. However, it is hardcoded to 0. 

### Root Cause

In `BlueprintCore:373`, index is 0.

### Internal Pre-conditions

None.

### External Pre-conditions

None.

### Attack Path

1. User creates an agent for a certain projectId, base64Proposal, server url.
2. User creates an agent (at the same block) with the same projectId, base64Proposal but different server url.
3. First request is overwritten.

### Impact

First request is overwritten and one of them will not be finalized as `submitProofOfDeployment()` and `submitDeploymentRequest()` can only be called once as part of the final steps by the worker. However, the user paid fees for both requests, but only one of them will go through.

### PoC

See above.

### Mitigation

Index should be increment in a user mapping.

## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/crestalnetwork/crestal-omni-contracts/pull/28

---
