# Cluster -1185

**Rank:** #36  
**Count:** 577  

## Label
Signature replay and misuse vulnerabilities enabling unauthorized transactions, impersonation, or front-running through insufficient message uniqueness, lack of sender validation, or inadequate input sanitization.

## Cluster Information
- **Total Findings:** 577

## Examples

### Example 1

**Auto Label:** Failure to properly bind signatures to chain ID and contract context enables replay attacks, signature reuse, and incorrect state validation across chains and transactions.  

**Original Text Preview:**

The [`deposit` function](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L189) of the `SpokePoolPeriphery` contract allows users to deposit native value to the SpokePool. However, its `recipient` and `exclusiveRelayer` arguments are both of type `address` and are [cast](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L209-L215) to `bytes32`. As a result, it is not possible to bridge wrapped native tokens to non-EVM blockchains.

Consider changing the type of the `recipient` and `exclusiveRelayer` arguments of the `deposit` function so that callers are allowed to specify non-EVM addresses for deposits.

***Update:** Resolved in [pull request #1018](https://github.com/across-protocol/contracts/pull/1018) at commit [`3f34af6`](https://github.com/across-protocol/contracts/pull/1018/commits/3f34af68b7602873e59a5a54b7c9cb0982f49d0e).*

---
### Example 2

**Auto Label:** Failure to properly bind signatures to chain ID and contract context enables replay attacks, signature reuse, and incorrect state validation across chains and transactions.  

**Original Text Preview:**

The [`PeripherySigningLib` library](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/libraries/PeripherySigningLib.sol#L6) contains the EIP-712 encodings of certain types as well as helper functions to generate their EIP-712 compliant hashed data. However, the [data type of the `SwapAndDepositData` struct](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/libraries/PeripherySigningLib.sol#L12-L13) is incorrect as it contains the `TransferType` member [of an enum type](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/interfaces/SpokePoolPeripheryInterface.sol#L18-L25), which is not supported by the EIP-712 standard.

Consider replacing the `TransferType` enum name used to generate the `SwapAndDepositData` struct's data type with `uint8` in order to be compliant with EIP-712.

***Update:** Resolved in [pull request #1017](https://github.com/across-protocol/contracts/pull/1017) at commit [`c9aaec6`](https://github.com/across-protocol/contracts/pull/1017/commits/c9aaec6d26993314e0bf878cd4eb89f447194789).*

---
### Example 3

**Auto Label:** Replay attacks via missing nonce checks or incomplete state inclusion in signatures, enabling signature reuse across transactions and compromising transaction integrity and order.  

**Original Text Preview:**

The [`SpokePoolPeriphery` contract](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L140) allows users to deposit or swap-and-deposit tokens into a SpokePool. In order to do that, the assets are first transferred from the depositor's account, optionally swapped to a different token, and then finally deposited into a SpokePool.

Assets can be transferred from the depositor's account in several different ways, including approval followed by the [`transferFrom` call](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L236-L240), [approval through the ERC-2612 `permit` function followed by `transferFrom`](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L267-L268), [transfer through the `Permit2` contract](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L295-L302), and [transfer through the ERC-3009 `receiveWithAuthorization` function](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L328-L338). The last three methods require additional user signatures and may be executed by anyone on behalf of a given user. However, the [data to be signed for deposits or swaps and deposits](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/interfaces/SpokePoolPeripheryInterface.sol#L70-L105) with ERC-2612 `permit` and with ERC-3009 `receiveWithAuthorization` does not contain a nonce, and, as such, the signatures used for these methods once can be replayed later.

The attack can be performed if a victim signs data for a function relying on the ERC-2612 `permit` function and wants to deposit tokens once again using the same method and token [within the time window determined by the `depositQuoteTimeBuffer` parameter](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePool.sol#L1306-L1307). In such a case, an attacker can first approve tokens on behalf of the victim and then call the [`swapAndBridgeWithPermit` function](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L249) or the [`depositWithPermit` function](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L357), providing a signature for a deposit or swap-and-deposit from the past, that includes fewer tokens than the approved amount.

As a result, the tokens will be deposited and potentially swapped, using the data from an old signature, forcing the victim to either perform an unintended swap or bridge the tokens to a different chain than intended. Furthermore, since the attack consumes some part of the `permit` approval, it will not be possible to deposit tokens on behalf of a depositor using the new signature until the full amount of tokens is approved by them once again. A similar attack is also possible in the case of functions that rely on the ERC-3009 `receiveWithAuthorization` function, but it requires the amount of tokens being transferred to be identical to the amount from the past.

Consider adding a nonce field into the [`SwapAndDepositData` and `DepositData` structs](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/interfaces/SpokePoolPeripheryInterface.sol#L70-L105) and storing a nonce for each user in the `SpokePoolPeriphery` contract, which should be incremented when a signature is verified and accepted.

***Update:** Resolved in [pull request #1015](https://github.com/across-protocol/contracts/pull/1015). The Across team has added a `permitNonces` mapping and extended both `SwapAndDepositData` and `DepositData` with a `nonce` field. In `swapAndBridgeWithPermit` and `depositWithPermit`, the contract now calls `_validateAndIncrementNonce(signatureOwner, nonce)` before verifying the EIP-712 signature, ensuring each permit-based operation can only be executed once. ERC-3009 paths continue to rely on the token’s own nonce; a replay here would require a token to implement both ERC-2612 and ERC-3009, a user to reuse the exact same nonce in both signatures, and both are executed within the narrow `fillDeadlineBuffer`. Given the unlikely convergence of these conditions, the risk is negligible in practice.*

---
