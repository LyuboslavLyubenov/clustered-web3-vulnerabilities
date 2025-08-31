# Cluster -1047

**Rank:** #374  
**Count:** 14  

## Label
Failure to handle invalid or edge cases with immediate reverts leads to silent failures, state corruption, or exploitable runtime errors, enabling griefing, DoS, or unauthorized operations.

## Cluster Information
- **Total Findings:** 14

## Examples

### Example 1

**Auto Label:** Failure to handle errors or edge cases leads to chain disruption, silent state corruption, or uncontrolled execution—enabling denial-of-service, data integrity loss, or consensus freezing.  

**Original Text Preview:**

## Summary

The function `isOperationDone(predecessor)` only checks if the predecessor operation was executed.

* However, if an operation is scheduled but later canceled, `executeBatch()` does not check if the predecessor was canceled.
* This allows a malicious proposer to create fake dependencies that will never resolve, blocking execution indefinitely.

## Vulnerability Details

* Attacker schedules an operation (Operation A) that acts as predecessor to another operation  (Operation B).
* They then schedule Operation B making it dependent on Operation A
* Before executing, they cancel Operation A (which is not checked in `executeBatch()`)
* Operation B remains blocked forever because it is waiting for Operation A to be executed, but it was canceled instead.
* This causes denial of service (DoS) for any operation that depends on the canceled predecessor.

  The same issue in `executeEmergencyAction()`

## Impact

* Predecessor Verification bypass
* Denial of Service (DoS)
* Unresolved Dependencies

## Tools Used

Manual Review

## Recommendations

Modify `isOperationDone()` to include Canceled state

Example implementation:

```Solidity
struct Operation {
uint64 timestamp;
bool executed;
bool canceled; // New flag to track cancellations
}
```

---
### Example 2

**Auto Label:** Failure to handle invalid or edge cases with immediate reverts leads to silent failures, state corruption, or exploitable runtime errors, enabling griefing, DoS, or unauthorized operations.  

**Original Text Preview:**

## Context
- **Bridge.sol**: Line 234
- **WERC721.sol**: Line 22

## Description
Per EIP-721, the methods `name()`, `symbol()`, and `tokenURI()` are optional extension functions. As such, some valid ERC-721 tokens do not support this interface and are incompatible with the Sweep n' Flip protocol.

## Recommendation
Consider using the `try/catch` mechanism to query these values and handle them accordingly.

## Sweep n' Flip
For the `WERC721.sol` file, `name()`/`symbol()` calls will only fail if they also fail for the wrapped collection. That is the intended behavior.

## Cantina Managed
Acknowledged. It should be added in the documentation that only contracts to implement the three methods: `name()`, `symbol()`, and `tokenURI()` are compatible with the protocol.

---
### Example 3

**Auto Label:** Failure to handle invalid or edge cases with immediate reverts leads to silent failures, state corruption, or exploitable runtime errors, enabling griefing, DoS, or unauthorized operations.  

**Original Text Preview:**

## ERC-721 Compatibility with Sweep n' Flip

## Context
- **Bridge.sol**: Line 234
- **WERC721.sol**: Line 22

## Description
Per EIP-721, the methods `name()`, `symbol()`, and `tokenURI()` are optional extension functions. As such, some valid ERC-721 tokens do not support this interface and are incompatible with the Sweep n' Flip protocol.

## Recommendation
Consider using the `try/catch` mechanism to query these values and handle them accordingly.

## Sweep n' Flip
For the `WERC721.sol` file, `name()`/`symbol()` calls will only fail if they also fail for the wrapped collection. That is the intended behavior.

## Cantina Managed
Acknowledged. It should be added in the documentation that only contracts implementing the three methods: `name()`, `symbol()`, and `tokenURI()` are compatible with the protocol.

---
