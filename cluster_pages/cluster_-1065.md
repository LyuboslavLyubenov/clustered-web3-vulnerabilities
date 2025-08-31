# Cluster -1065

**Rank:** #6  
**Count:** 1038  

## Label
Insufficient input validation and state transition flaws enable attackers to exploit financial invariants, cause unintended portfolio exposure, or manipulate auction outcomes through bid re-execution, zero-value bids, or unbounded challenge cycles.

## Cluster Information
- **Total Findings:** 1038

## Examples

### Example 1

**Auto Label:** Insufficient input validation allows malicious actors to assign invalid addresses or exceed bounds, leading to unintended state changes, fund loss, or exploitable transaction flows.  

**Original Text Preview:**

When operations with address parameters are performed, it is crucial to ensure the address is not set to zero. Setting an address to zero is problematic because it has special burn/renounce semantics. This action should be handled by a separate function to prevent accidental loss of access during value or ownership transfers.

Throughout the codebase, there are multiple instances where operations are missing a zero address check:

* The [`_setMessenger(messengerType, dstDomainId, srcChainToken, srcChainMessenger)`](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/contracts/AdapterStore.sol#L34) operation within the contract `AdapterStore` in `AdapterStore.sol`.
* The [`_setMessenger(messengerTypes[i], dstDomainIds[i], srcChainTokens[i], srcChainMessengers[i])`](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/contracts/AdapterStore.sol#L52) operation within the contract `AdapterStore` in `AdapterStore.sol`.
* The [`_adapterStore`](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/contracts/libraries/OFTTransportAdapterWithStore.sol#L17) operation within the contract `OFTTransportAdapterWithStore` in `OFTTransportAdapterWithStore.sol`.
* The [`_setOftMessenger(token, messenger)`](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/contracts/SpokePool.sol#L363) operation within the contract `SpokePool` in `SpokePool.sol`.

Consider adding a zero address check before assigning a state variable.

***Update:** Acknowledged, not resolved. The team stated:*

> *After internal discussion, we think that zero-address checks here are a bit of overkill because all the functions mentioned are only callable by admin (except for `_adapterStore` case, but that's contract creation, and this contract can only be used within the system after admin action)*

---
### Example 2

**Auto Label:** Insufficient input validation leads to unauthorized operations, incorrect routing, or unintended token transfers, enabling denial-of-service, fund loss, or malicious redirection.  

**Original Text Preview:**

The [`SwapProxy` contract](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L26) contains the [`performSwap` function](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L67), which allows the caller to execute a swap in two ways: [by approving or sending tokens to the specified exchange](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L81-L84), or by [approving tokens through the `Permit2` contract](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L86-L101). However, since it is possible to supply any address as the `exchange` parameter and any call data through the `routerCalldata` parameter of the `performSwap` function, the `SwapProxy` contract may be forced to perform an [arbitrary call to an arbitrary address](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L108).

This could be exploited by an attacker, who could force the `SwapProxy` contract to call the [`invalidateNonces` function](https://github.com/Uniswap/permit2/blob/cc56ad0f3439c502c246fc5cfcc3db92bb8b7219/src/AllowanceTransfer.sol#L113) of the `Permit2` contract, specifying an arbitrary spender and a nonce higher than the current one. As a result, the nonce for the given (token, spender) pair will be [updated](https://github.com/Uniswap/permit2/blob/cc56ad0f3439c502c246fc5cfcc3db92bb8b7219/src/AllowanceTransfer.sol#L124). If the `performSwap` function is called again later, it [will attempt to use a subsequent nonce](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L95), which has been invalidated by the attacker and the code inside `Permit2` will [revert due to nonces mismatch](https://github.com/Uniswap/permit2/blob/cc56ad0f3439c502c246fc5cfcc3db92bb8b7219/src/AllowanceTransfer.sol#L138).

As the `performSwap` function is the only place where the nonce passed to the `Permit2` contract is updated, the possibility of swapping a given token on a certain exchange will be blocked forever, which impacts all the functions of the [`SpokePoolPeriphery` contract](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L140) related to swapping tokens. The attack may be performed for many different (tokens, exchange) pairs.

Consider not allowing the `exchange` parameter to be equal to the `Permit2` contract address.

***Update:** Resolved in [pull request #1016](https://github.com/across-protocol/contracts/pull/1016) at commit [`713e76b`](https://github.com/across-protocol/contracts/pull/1016/commits/713e76b8388d90b4c3fbbe3d16b531d3ef81c722).*

---
### Example 3

**Auto Label:** Missing input or state validation leads to exploitable fee or parameter manipulation, enabling incorrect logic, overflow, or unfair economic outcomes.  

**Original Text Preview:**

In the `updateFeeToken`, `updateFeeAmount`, and `updateFeeTo` functions, the fee config is not checked to ensure it has been properly set beforehand. As a result, an editor could mistakenly update the fee config for a chain ID where the fee config has not been set.

Example of impact: If the editor updates the fee token, the fee config could not be set for that chain ID.

---
