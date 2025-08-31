# Cluster -1038

**Rank:** #142  
**Count:** 99  

## Label
Inconsistent state validation and access controls across contracts lead to unauthorized access, state mismatches, and exploitable behavior due to missing or misaligned parameter checks and invariant enforcement.

## Cluster Information
- **Total Findings:** 99

## Examples

### Example 1

**Auto Label:** Failure to validate critical state invariants during execution flows, enabling race conditions, unchecked approvals, or invalid state transitions that compromise asset integrity and protocol security.  

**Original Text Preview:**

## Severity: Low Risk

## Context
BaseVault.sol#L126-L130

## Description
In the `_executeSubmit()` function, `_noPendingApprovalsInvariant()` is executed before the `_afterSubmitHooks()` call. Therefore, if the after-submit hook makes a callback to the vault to execute `approve()`, that approval will be unchecked. 

This scenario is theoretically possible because the vault does not check if a registered callback has been received during the operation, so the callback can still be executed after the operation. The callback would need to pass the caller and selector checks, so the situation would likely be a configuration error and compromise of the hook or target contract.

## Recommendation
While the above scenario is unlikely to occur, to be on the safe side, consider checking `_noPendingApprovalsInvariant()` after `_afterSubmitHooks()` (i.e., at the end of the `submit()` function) to explicitly guarantee that no approvals are left at the end of the submission.

## Aera
Fixed in PR 218.

## Spearbit
Verified.

---
### Example 2

**Auto Label:** Failure to validate critical state invariants during execution flows, enabling race conditions, unchecked approvals, or invalid state transitions that compromise asset integrity and protocol security.  

**Original Text Preview:**

## Severity: Medium Risk

**Context:** `OracleRegistry.sol#L187-L198`

**Description:**  
The vault owners could accept an unexpected oracle through the `acceptPendingOracle()` function. 

**Consider the following scenario:**

1. The protocol owner schedules an update.
2. The vault owner sees the update and sends a transaction to accept it.
3. Before the transaction is executed, the protocol owner cancels the update and schedules another one.
4. The vault owner ends up accepting an oracle they do not expect.

Eventually, the vault owner will still use the new oracle after the update delay has passed. However, the vault owner may want to take enough time to review the new oracle before accepting it in case of security risks.

**Recommendation:**  
Consider modifying the `acceptPendingOracle()` function so that it takes a parameter `pendingOracle`, which is compared with the on-chain `pendingOracleData[base][quote]` value. If the two values are different, the function should revert.

**Aera:** Fixed in PR 289.  
**Spearbit:** Verified.

---
### Example 3

**Auto Label:** Failure to validate critical state invariants during execution flows, enabling race conditions, unchecked approvals, or invalid state transitions that compromise asset integrity and protocol security.  

**Original Text Preview:**

## Risk Assessment

## Severity 
**Medium Risk**

## Context 
**File:** OracleRegistry.sol  
**Lines:** L73-L89  

## Description  
Adding an oracle through the `addOracle()` function makes the oracle effective immediately. There should be a delay period for vault owners to review the oracle in case the protocol gets compromised and a malicious oracle is added. This would only affect the case where the vault has already been configured to allow operations based on the base/quote price before the oracle is added. The case is rare but not impossible for some vault owners to do so.

## Recommendation  
Consider introducing a delay when a new oracle is added so the vault owners have enough time to review it. A possible change could be to remove the `require(currentOracle.isNotEmpty(), ...)` check in the `scheduleOracleUpdate()` function so that it also works for adding new oracles.

## Aera 
Unfortunately, this could make it tricky to create new strategies quickly, so while we can't remove `addOracle`, we will check that the oracle exists when adding a new token in `Provisioner setTokenDetails`. Fixed in PR 302.

## Spearbit 
Verified.

---
