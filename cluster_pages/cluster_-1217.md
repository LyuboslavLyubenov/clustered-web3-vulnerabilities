# Cluster -1217

**Rank:** #317  
**Count:** 22  

## Label
Failure to validate or check contract state before execution leads to incorrect state updates, unauthorized operations, or silent failures, risking asset loss, inconsistent accounting, or exploitation during invalid or transitional states.

## Cluster Information
- **Total Findings:** 22

## Examples

### Example 1

**Auto Label:** Failure to validate or verify recipient/authority/contract existence leads to unauthorized delegation, asset mismanagement, or incorrect execution, enabling loss of control or irreversible protocol errors.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Low

## Description

The `_bridge()` and `_quote()` in the `MultiChainHyperlaneTellerWithMultiAssetSupport` contract fail to apply a custom post-dispatch hook set via the `setHook()`. Instead, they rely on the overloaded `Mailbox.dispatch()`, which defaults to using the Mailbox contract's defaultHook.

```solidity
function _bridge(uint256 shareAmount, BridgeData calldata data) internal override returns (bytes32 messageId) {
--- SNIPPED ---
    mailbox.dispatch{ value: msg.value }(
        data.chainSelector, // must be `destinationDomain` on hyperlane
        msgRecipient, // must be the teller address left-padded to bytes32
        _payload,
        StandardHookMetadata.overrideGasLimit(data.messageGas) // Sets the refund address to msg.sender, sets
            // `_msgValue`
            // to zero
    );
}
```

The `Mailbox.dispatch()` has three overloaded variants. In this case, the `_bridge()` invokes the version without specifying a custom IPostDispatchHook, causing the `Mailbox` contract to automatically apply with its `defaultHook`.

```solidity
// hyperlane-monorepo/solidity/contracts/Mailbox.sol
function dispatch(
    uint32 destinationDomain,
    bytes32 recipientAddress,
    bytes calldata messageBody,
    bytes calldata hookMetadata
) external payable override returns (bytes32) {
    return
        dispatch(
            destinationDomain,
            recipientAddress,
            messageBody,
            hookMetadata,
@>          defaultHook
        );
}
```

Consequently, the `MultiChainHyperlaneTellerWithMultiAssetSupport` opts to use a custom post-dispatch hook instead of the `Mailbox` contract's default, the custom hook is not invoked during the dispatch process. This could result in incorrect or unintended post-dispatch handling.

```solidity
// hyperlane-monorepo/solidity/contracts/Mailbox.sol
function dispatch(
    uint32 destinationDomain,
    bytes32 recipientAddress,
    bytes calldata messageBody,
    bytes calldata metadata,
    IPostDispatchHook hook
) public payable virtual returns (bytes32) {
    if (address(hook) == address(0)) {
        hook = defaultHook;
    }
--- SNIPPED ---
    requiredHook.postDispatch{value: requiredValue}(metadata, message);
@>  hook.postDispatch{value: msg.value - requiredValue}(metadata, message);

    return id;
}
```

Note that this issue also arises with the `_quote()` that can lead to incorrect fee estimation.

## Recommendations

Consider using the overloaded `dispatch()` method in the `Mailbox` contract that accepts a custom hook.

This approach will handle both cases where a hook is set and not set because it will also default to using the `defaultHook` if the protocol decides to leave the hook as the zero address.

Please note that the custom hooks should be compatible with the Hyperlane standard hook.

```diff
function _bridge(uint256 shareAmount, BridgeData calldata data) internal override returns (bytes32 messageId) {
--- SNIPPED ---
    mailbox.dispatch{ value: msg.value }(
        data.chainSelector, // must be `destinationDomain` on hyperlane
        msgRecipient, // must be the teller address left-padded to bytes32
        _payload,
-       StandardHookMetadata.overrideGasLimit(data.messageGas) // Sets the refund address to msg.sender, sets
-            // `_msgValue`
-            // to zero
+       StandardHookMetadata.overrideGasLimit(data.messageGas),
+       hook        
    );
}
```

---
### Example 2

**Auto Label:** Failure to validate or verify recipient/authority/contract existence leads to unauthorized delegation, asset mismanagement, or incorrect execution, enabling loss of control or irreversible protocol errors.  

**Original Text Preview:**

##### Description
This issue has been identified within the `_delegateStake` function of the `HookTargetStakeDelegator` contract.
The function directly calls `rewardVault.delegateStake(...)` without first verifying whether the `rewardVault` contract has been successfully deployed. In edge cases such as when deployment is asynchronous or delayed this may lead to unexpected reverts and disrupt protocol execution.
<br/>
##### Recommendation
We recommend adding a check to confirm the existence of the `rewardVault` contract (e.g., using `extcodesize`) before attempting to delegate stake. If the contract is not yet deployed, the function should simply skip delegation to maintain protocol continuity.

> **Client's Commentary:**
> Fixed: https://github.com/euler-xyz/evk-periphery/pull/274/files

---

---
### Example 3

**Auto Label:** Failure to validate or verify recipient/authority/contract existence leads to unauthorized delegation, asset mismanagement, or incorrect execution, enabling loss of control or irreversible protocol errors.  

**Original Text Preview:**

## Summary

The `delegateBoost()` function **does not validate the recipient (`to`) properly**, allowing users to **delegate boosts to unverified addresses**. This could lead to **boosts being sent to malicious contracts, ineffective delegations, or self-delegations**, breaking the system’s integrity.

## Vulnerability Details

Vulnerable Code:

```Solidity
function delegateBoost(
    address to,
    uint256 amount,
    uint256 duration
) external override nonReentrant {
    if (paused()) revert EmergencyPaused();
    if (to == address(0)) revert InvalidPool();  // Only checks for `0x0`
    if (amount == 0) revert InvalidBoostAmount();
    if (duration < MIN_DELEGATION_DURATION || duration > MAX_DELEGATION_DURATION) 
        revert InvalidDelegationDuration();

    uint256 userBalance = IERC20(address(veToken)).balanceOf(msg.sender);
    if (userBalance < amount) revert InsufficientVeBalance();

    // No verification if `to` is a valid recipient
}

```

**Problems**

* **A user can delegate to an unverified or malicious contract**, which could **steal or misuse the boost**.
* **No check to prevent self-delegation**, leading to **ineffective boosts**.
* **Boosts can be locked in contracts that do not support retrieval**, permanently losing voting power.

## Impact

* **Boosts can be lost forever** if sent to a **malicious contract or an invalid address**.

- **Users could self-delegate, causing ineffective boost calculations.**
- **Boost system integrity is compromised, allowing abuse.**

## Tools Used

Manual review

## Recommendations

* **Ensure that** **`to`** **is a valid pool using** **`supportedPools[to]`.**

- **Prevent self-delegation.**

```Solidity
function delegateBoost(
    address to,
    uint256 amount,
    uint256 duration
) external override nonReentrant {
    if (paused()) revert EmergencyPaused();
    if (!supportedPools[to]) revert PoolNotSupported();  // Ensure recipient is a valid pool
    if (msg.sender == to) revert CannotSelfDelegate();  // Prevent self-delegation

    // ..
}

```

---
