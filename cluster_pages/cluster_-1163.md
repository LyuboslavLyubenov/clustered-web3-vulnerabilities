# Cluster -1163

**Rank:** #407  
**Count:** 11  

## Label
Poor control flow design leads to inconsistent state management, hard-to-maintain logic, and unintended behavior under varying conditions, increasing vulnerability to bugs and attack surface expansion.

## Cluster Information
- **Total Findings:** 11

## Examples

### Example 1

**Auto Label:** Poor control flow design leads to inconsistent state management, hard-to-maintain logic, and unintended behavior under varying conditions, increasing vulnerability to bugs and attack surface expansion.  

**Original Text Preview:**

**Description:** The `LogicalOrWrapperEnforcer` contract currently includes the `onlyDefaultExecutionMode` modifier on all of its hook functions. This creates an unnecessary restriction since the individual enforcers being wrapped already apply their own execution mode restrictions. This design decision could limit the wrapper's flexibility and prevent it from working with caveats that might support non-default execution modes.


**Recommended Mitigation:** Consider removing the `onlyDefaultExecutionMode` modifier from all hook functions in the `LogicalOrWrapperEnforcer` and let the individual wrapped caveat enforcers handle their own execution mode restrictions.

Not only is this gas efficient, this change would also allow the `LogicalOrWrapperEnforcer` to be more flexible and forward-compatible with future caveats, while still maintaining the appropriate execution mode restrictions through the wrapped enforcer contracts themselves.

**Metamask:** Fixed in commit [d38d53d](https://github.com/MetaMask/delegation-framework/commit/d38d53dc467cc3b4faa7047cfca1844ea9cbc3be).

**Cyfrin:** Resolved.

---
### Example 2

**Auto Label:** Misuse of conditional logic leading to unintended state transitions, flawed control flow, and ambiguous execution paths that enable unauthorized actions or undefined behavior.  

**Original Text Preview:**

## Context
- TokenFlow.sol#L12-L13
- TokenFlow.sol#L35-L43
- TransientNetflows.sol#L6

## Description/Recommendation

1. **TransientNetflows.sol#L6**  
   Typo: "An helper" should be "A helper".

2. **TokenFlow.sol#L12-L13**  
   Visibility is missing for both constants; consider declaring them private:
   - ```solidity
     uint constant EXTERNAL_SCOPE = 0;
     uint constant INTERNAL_SCOPE = 1;
     ```
   + ```solidity
     uint256 constant private EXTERNAL_SCOPE = 0;
     uint256 constant private INTERNAL_SCOPE = 1;
     ```

3. **TokenFlow.sol#L35-L43**  
   The `main()` function performs a low-level call to `internalScope` when calling `enter()`.  
   However, since `abi.encodeCall()` is used and the logic in `main()` reverts on failure, a high-level call can be used instead:
   - ```solidity
     (bool ok, bytes memory err) = address(internalScope).call(abi.encodeCall(IFlowScope.enter,
     (SELECTOR_EXTENSION, constraints, msg.sender, data)));
     //...
     if (!ok) {
         clearTransientState();
         // bubble up the error
         assembly {
             revert(add(err, 0x20), mload(err))
         }
     }
     ```
   + ```solidity
     IFlowScope(internalScope).enter(SELECTOR_EXTENSION, constraints, msg.sender, data);
     ```
   The only difference is that when using a low-level call, `main()` does not revert if `internalScope` is an address without code. However, this is not desired behavior.

## Flood
All issues have been fixed in commit e434c000.

## Cantina Managed
Verified, all recommendations have been implemented.

---
### Example 3

**Auto Label:** Misuse of conditional logic leading to unintended state transitions, flawed control flow, and ambiguous execution paths that enable unauthorized actions or undefined behavior.  

**Original Text Preview:**

**Severity**

**Impact:** High

**Likelihood:** Medium

**Description**

Pools are designed to be paused, unpaused or killed. If killed, they cannot then be unpaused again.

Lets look at the `unpause` function.

```solidity
 if (!paused) revert Pool__NotPaused();
if (!killed) revert Pool__Killed();
paused = false;
```

The second line checks the killed status. It basically says that if the pool is not killed, the function should revert. This is a logical flaw. The function should revert IF the pool IS killed, not otherwise. The pool `killed` variable starts out as false. Thus if a pool is paused and then unpaused, the unpause will not happen since `!false=true`, so the revert will trigger.

A similar issue is present in the `kill` function.

```solidity
if (!killed) revert Pool__Killed();
        killed = true;
```

Again, if the contract is already killed this should revert. In the current state, the contract can never be killed, since `killed` is set to false by default, so `!killed` will always be true, making this code unreachable.

**Recommendations**

In both cases, change the clause to `if(killed)`

---
