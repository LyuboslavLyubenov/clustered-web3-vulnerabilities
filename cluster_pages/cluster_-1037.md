# Cluster -1037

**Rank:** #373  
**Count:** 14  

## Label
Race conditions in subscription management due to out-of-order state updates, allowing subscribers to remain active during callbacks and exploit system state.

## Cluster Information
- **Total Findings:** 14

## Examples

### Example 1

**Auto Label:** Race conditions and state inconsistency in hook lifecycle management due to improper synchronization, state persistence, and execution ordering.  

**Original Text Preview:**

## Risk Assessment Report

## Severity
**Medium Risk**

## Context
`CCTPHooks.sol#L17-L27`

## Description
Based on the tests, the CCTP hook is a before-hook but not an after-hook, which means that the `depositForBurn()` function will only be called once before the operation. Therefore, the `HAS_BEFORE_BEEN_CALLED_SLOT` remains 1 after the operation (i.e., not reset to 0).

Consider a case where the guardian wants to bridge USDC to multiple recipients in a submission. The second call
to `depositForBurn()` will fail since the hook considers that it is in a post-op state and returns empty bytes, which causes the Merkle proof verification to fail.

## Recommendation
Consider either removing the `if (HooksLibrary.hasBeforeHookBeenCalled())` statement if the CCTP hook is a before-hook but not an after-hook. Otherwise, configure the CCTP hook as both a before- and after-hook so the slot can be reset to 0 after the operation. Generally speaking, if a hook uses the `hasBeforeHookBeenCalled()` function, it should be both a before- and after-hook.

## Aera
Fixed in PR 303.

## Spearbit
Verified. The CCTP hook is now only a before-hook, but not an after-hook.

---
### Example 2

**Auto Label:** Race conditions and state inconsistency in hook lifecycle management due to improper synchronization, state persistence, and execution ordering.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

Within the `runExecutionHooks` modifier, it iterates through all registered execution hooks, triggers the `preExecutionHook`, and sets the hook's `context`. Then, at the end of execution, it iterates again, triggers the `postExecutionHook` if the `context` exists, and finally deletes the context.

```solidity
    modifier runExecutionHooks(Transaction calldata transaction) {
        mapping(address => address) storage executionHooks = _executionHooksLinkedList();

        address cursor = executionHooks[AddressLinkedList.SENTINEL_ADDRESS];
        // Iterate through hooks
        while (cursor > AddressLinkedList.SENTINEL_ADDRESS) {
            // Call the preExecutionHook function with transaction struct
            bytes memory context = IExecutionHook(cursor).preExecutionHook(transaction);
            // Store returned data as context
            _setContext(cursor, context);

            cursor = executionHooks[cursor];
        }

        _;

        cursor = executionHooks[AddressLinkedList.SENTINEL_ADDRESS];
        // Iterate through hooks
        while (cursor > AddressLinkedList.SENTINEL_ADDRESS) {
            bytes memory context = _getContext(cursor);
            if (context.length > 0) {
                // Call the postExecutionHook function with stored context
                IExecutionHook(cursor).postExecutionHook(context);
                // Delete context
                _deleteContext(cursor);
            }

            cursor = executionHooks[cursor];
        }
    }
```

It is possible that during the execution of a function with the `runExecutionHooks` modifier calls the `removeHook` and remove one of the registered hooks. However, since the removed hook is no longer registered in `executionHooks`, its context will not be removed at the end of `runExecutionHooks` modifier's execution.

This could lead to unexpected behavior, especially if the removed hook is re-added in the future, as the old context could be used during some operations.

## Recommendations

Remove the hook's context when `removeHook` is called.

---
### Example 3

**Auto Label:** Race conditions caused by improper synchronization and inconsistent receiver types in concurrent access to shared state.  

**Original Text Preview:**

## Risk Assessment Report

## Severity
**Low Risk**

## Context
- `abci.go`: Lines 58-71
- `keeper.go`: Lines 206-211

## Description
In `PrepareProposal()`, the `getOptimisticPayload()` method returns a pointer to the payload identifier. When we finally get the payload, we use the pointer to assign the identifier. The mutable payload has a lock and we are writing to it without holding a write lock on it. Out of caution, we should avoid this pattern.

## Recommendation
1. Maybe use `setOptimisticPayload()` to assign the payload ID.
2. Change `ID` in `mutablePayload` to a non-pointer type; this is prone to errors.

## Omni
Fixed in commit `60904774` by changing `ID` to a non-pointer type.

## Spearbit
Fix verified.

---
