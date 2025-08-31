# Cluster -1056

**Rank:** #152  
**Count:** 91  

## Label
Unauthorized access and validation bypasses in cross-chain message handling lead to unauthorized operations, fund misuse, and transaction loss due to insufficient sender, caller, and message state checks.

## Cluster Information
- **Total Findings:** 91

## Examples

### Example 1

**Auto Label:** Unauthorized access and validation bypasses in cross-chain message handling lead to unauthorized operations, fund misuse, and transaction loss due to insufficient sender, caller, and message state checks.  

**Original Text Preview:**

Inside `MultiChainTellerBase`, there are several operations to allow or stop receiving messages from a configured chain (`allowMessagesFrom` / `allowMessagesTo`). However, it does not verify if `targetTeller` is a non-empty address when `allowMessagesFrom` / `allowMessagesTo` is set to true. This could cause issues, as `mailbox.dispatch` will set `msgRecipient` to an empty address if misconfigured. Consider adding additional verification.

---
### Example 2

**Auto Label:** Misuse of `msg.sender` in multicall contexts leads to incorrect sender identity, enabling unauthorized access, logic errors, and MEV exploitation due to failure to track original transaction originators.  

**Original Text Preview:**

**Description:** `LibMulticaller` is used throughout the codebase to retrieve the actual sender of multicall transactions; however, `BunniToken::_beforeTokenTransfer` and `BunniToken::_afterTokenTransfer` both incorrectly pass `msg.sender` directly to the corresponding Hooklet function:

```solidity
function _beforeTokenTransfer(address from, address to, uint256 amount, address newReferrer) internal override {
    IHooklet hooklet_ = hooklet();
    if (hooklet_.hasPermission(HookletLib.BEFORE_TRANSFER_FLAG)) {
        hooklet_.hookletBeforeTransfer(msg.sender, poolKey(), this, from, to, amount);
    }
}

function _afterTokenTransfer(address from, address to, uint256 amount) internal override {
    // call hooklet
    IHooklet hooklet_ = hooklet();
    if (hooklet_.hasPermission(HookletLib.AFTER_TRANSFER_FLAG)) {
        hooklet_.hookletAfterTransfer(msg.sender, poolKey(), this, from, to, amount);
    }
}
```

**Impact:** The Hooklet calls will reference the incorrect sender. This has potentially serious downstream effects for integrators as custom logic is executed with the incorrect address in multicall transactions.

**Recommended Mitigation:** `LibMulticaller.senderOrSigner()` should be used in place of `msg.sender` wherever the actual sender is required:

```diff
function _beforeTokenTransfer(address from, address to, uint256 amount, address newReferrer) internal override {
    IHooklet hooklet_ = hooklet();
    if (hooklet_.hasPermission(HookletLib.BEFORE_TRANSFER_FLAG)) {
--      hooklet_.hookletBeforeTransfer(msg.sender, poolKey(), this, from, to, amount);
++      hooklet_.hookletBeforeTransfer(LibMulticaller.senderOrSigner(), poolKey(), this, from, to, amount);
    }
}

function _afterTokenTransfer(address from, address to, uint256 amount) internal override {
    // call hooklet
    IHooklet hooklet_ = hooklet();
    if (hooklet_.hasPermission(HookletLib.AFTER_TRANSFER_FLAG)) {
--      hooklet_.hookletAfterTransfer(msg.sender, poolKey(), this, from, to, amount);
++      hooklet_.hookletAfterTransfer(LibMulticaller.senderOrSigner(), poolKey(), this, from, to, amount);
    }
}
```

**Bacon Labs:** Fixed in [PR \#106](https://github.com/timeless-fi/bunni-v2/pull/106).

**Cyfrin:** Verified, the `LibMulticaller` is now used to pass the `msg.sender` in `BunniToken` Hooklet calls.

---
### Example 3

**Auto Label:** Unauthorized access and validation bypasses in cross-chain message handling lead to unauthorized operations, fund misuse, and transaction loss due to insufficient sender, caller, and message state checks.  

**Original Text Preview:**

The `ForwarderBase` contract has the `setCrossDomainAdmin` [function](https://github.com/across-protocol/contracts/blob/5a0c67c984d19a3bb843a4cec9bb081734583dd1/contracts/chain-adapters/ForwarderBase.sol#L69) which changes the `crossDomainAdmin` state [variable](https://github.com/across-protocol/contracts/blob/5a0c67c984d19a3bb843a4cec9bb081734583dd1/contracts/chain-adapters/ForwarderBase.sol#L20). This variable is used to give access control to almost all functions within the contract, including those that are meant to be used to complete L1->L3 message passing and asset transfers. When calling `setCrossDomainAdmin`, it is assumed that there are no outstanding operations that need to be passed to another chain. This is because when operations arrive at L2, the original sender of those is the old `crossDomainAdmin` and will be blocked from continuing further.

In order to raise awareness about such behavior, consider documenting this edge case in the `setCrossDomainAdmin` function.

***Update:** Resolved in [pull request #729](https://github.com/across-protocol/contracts/pull/729) at commit [4ba3439](https://github.com/across-protocol/contracts/pull/729/commits/4ba34394de24cf79cdce84d9ceccfe61e2b21d03).*

---
