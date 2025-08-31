# Cluster -1323

**Rank:** #212  
**Count:** 52  

## Label
Missing validation of zero addresses in governance functions enables attackers to disable critical operations, leading to loss of control and system failure through invalid or unauthorized configuration.

## Cluster Information
- **Total Findings:** 52

## Examples

### Example 1

**Auto Label:** Failure to validate or restrict critical address or state updates leads to irreversible control loss or unauthorized contract takeover.  

**Original Text Preview:**

## Data Validation Report

## Difficulty: Low

## Type: Data Validation

### File Location
`nominators-v2/src/contract/nominators.fc`

### Description
When called, the `op::change_address` operation immediately sets the owner or controller address to a new value. The use of a single step to make such a critical change is error-prone; if the function is called with erroneous input, the results could be irrevocable, leading to permanent loss of important owner functionality.

```fc
() op_change_address(int value, slice in_msg) impure {
    ;; Parse parameters
    var id = in_msg~load_uint(8);
    var new_address = in_msg~load_msg_addr();
    in_msg.end_parse();
    
    ;; Throw if new_address is already owner or controller
    if (equal_slices(ctx_owner, new_address) | equal_slices(ctx_controller, new_address)) {
        throw(error::invalid_message());
    }
    
    ;; Update address
    if (id == 0) {
        ctx_owner = new_address;
    } elseif (id == 1) {
        ctx_controller = new_address;
    } else {
        throw(error::invalid_message());
    }
}
```
*Figure 2.1: The `op_change_address` function, callable by the contract owner (nominators-v2/src/contract/modules/op-owner.fc#L1–L20)*

### Recommendations
- **Short term:** Implement a two-step process for all irrevocable critical operations. The `treasure-v0` contract from the holders repository uses a two-step process for ownership transfer, which can be implemented in the nominators contract to resolve this issue.
  
- **Long term:** Identify and document all possible actions that can be taken by privileged accounts, along with their associated risks. This will facilitate reviews of the codebase and prevent future mistakes.

---
### Example 2

**Auto Label:** Missing validation of zero addresses in governance functions enables attackers to disable critical operations, leading to loss of control and system failure through invalid or unauthorized configuration.  

**Original Text Preview:**

**Severity**: Medium

**Status**: Resolved

**Description (Contracts & Functions)**:

EvoqGovernance.sol (e.g., setPositionsManager(), setTreasuryVault(), etc.)
Certain governance or admin functions allow setting critical addresses (like positionsManager, treasuryVault) to the zero address. This breaks protocol logic if a call to address(0) silently no-ops or otherwise fails unexpectedly, yet returns success.

**Scenario:**

An owner or multisig calls setPositionsManager(address(0)) by mistake.
The system’s subsequent calls to positionsManager become calls to address(0), effectively doing nothing or reverting unpredictably.
Operations dependent on positionsManager fail until corrected.

**Recommendation:**

Add require(_newAddress != address(0), "Zero address not allowed") in all setters for critical addresses.
Implement a timelock or thorough review step before finalizing critical parameter changes.

---
### Example 3

**Auto Label:** Missing validation of zero addresses in governance functions enables attackers to disable critical operations, leading to loss of control and system failure through invalid or unauthorized configuration.  

**Original Text Preview:**

## Code Review for LauncherPlugin.sol

## Context
**File:** LauncherPlugin.sol  
**Line:** 101

## Description
This is the first require statement of `migratePool`.

```solidity
require(
    msg.sender == address(voter) || msg.sender == operator,
    IVoter.NOT_AUTHORIZED(msg.sender)
);
```

As we can see, the only two addresses that can call this function are the operator and the Voter contract. In reality, the Voter contract never calls this function.

## Recommendation
Consider removing this permission to comply with the least authority concept.

## Shadow Exchange
Fixed in commit `6c3ca1cb`.

## Cantina Managed
Verified.

---
