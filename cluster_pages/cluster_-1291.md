# Cluster -1291

**Rank:** #104  
**Count:** 163  

## Label
Merkle proof validation flaws enabling unauthorized access, replay, or manipulation through missing bounds checks, chain identity verification, or function context differentiation.

## Cluster Information
- **Total Findings:** 163

## Examples

### Example 1

**Auto Label:** Merkle proof validation flaws enabling unauthorized access, replay, or manipulation through missing bounds checks, chain identity verification, or function context differentiation.  

**Original Text Preview:**

The functions [verify\_with\_builder](https://github.com/OpenZeppelin/rust-contracts-stylus/blob/a67ab7068ac94a5a1ad48ae632050c8d28f9ab25/lib/crypto/src/merkle.rs#L178) and [verify\_multi\_proof\_with\_builder](https://github.com/OpenZeppelin/rust-contracts-stylus/blob/a67ab7068ac94a5a1ad48ae632050c8d28f9ab25/lib/crypto/src/merkle.rs#L257) serve as verifiers for Merkle trees with custom hashing functions. However, these functions reconstruct the Merkle root in a commutative manner, which assumes that all custom hashing algorithms employ commutative hashing. Although commutative hashing is common practice, this assumption cannot be guaranteed for all custom hashing processes.

Consider updating the documentation to specify that the Merkle tree hashing process must be constructed commutatively when using custom hashing algorithms.

***Update:** Resolved in [pull #674](https://github.com/OpenZeppelin/rust-contracts-stylus/pull/674/files) at commit [6f75f80](https://github.com/OpenZeppelin/rust-contracts-stylus/commit/6f75f80ca00fe768bc53aafc7772c9b6ec526afb).*

---
### Example 2

**Auto Label:** Merkle proof validation flaws enabling unauthorized access, replay, or manipulation through missing bounds checks, chain identity verification, or function context differentiation.  

**Original Text Preview:**

## Security Risk Assessment

## Severity
**Medium Risk**

## Context
`BaseVault.sol#L547-L562`

## Description
There are two potential security risks in the current callback design:

1. An operation's callback data is not included in the Merkle leaf, and therefore, guardians can decide whether the vault will receive a callback during the operation. Hooks could be implemented with an assumption that no callback will be received during the operation, which, if violated, could lead to unexpected consequences. See Issue "Incorrect calculation of the received swap amount allows guardians to bypass the daily loss limit" for more details.

2. During a callback, the sub-operations to execute are decoded directly from calldata. If the target contract that the vault interacts with is compromised, an adversary may be able to perform arbitrary operations with the guardian's permission.

## Recommendation
To mitigate the above risks, consider implementing the following:

- If an operation has a callback, include a hash in the `callbackData` part. The hash is the `keccak256` result of the encoded sub-operations that will be executed during the callback. Store this value in transient storage in `_-allowCallback()`. When a callback is received, extract the encoded sub-operations from calldata, hash it, and compare it with this value. Revert with an error if the two values differ, which indicates that the sub-operations have been modified.

- Also, consider including `callbackData` in the Merkle leaf, e.g.,

```solidity
function _createMerkleLeaf(OperationContext memory ctx, bytes memory extractedData)
    internal
    pure
    returns (bytes32)
{
    return keccak256(
        abi.encodePacked(
            ctx.target,
            ctx.selector,
            ctx.value > 0,
            ctx.operationHooks,
            ctx.configurableOperationHooks,
            ctx.hasCallback ? keccak256(ctx.callbackData) : bytes32(0),
            extractedData
        )
    );
}
```

This would allow the vault owners to explicitly allow or disallow callbacks during an operation, and the sub-operations executed in the callback.

## Aera
We think it is too limiting to enforce sub-operations in the Merkle tree and violates the spirit of the protocol which is to empower guardians to optimize actions off-chain. However, we will implement the latter suggestion.

## Spearbit
To clarify, the hash of the encoded sub-ops does not have to be included in the Merkle tree, but just storing it in the storage and comparing it with the actual received sub-ops would be sufficient. That being said, since this check is mainly for the guardians to protect themselves, it is fine not to implement this in the contract to avoid complexity overhead. Guardians should always review the target contracts and/or simulate the transactions to ensure they are not interacting with potentially compromised contracts.

## Resolution
**Aera:** Fixed in PR 300.  
**Spearbit:** Verified.

---
### Example 3

**Auto Label:** Nonce mismanagement leading to replay attacks, validation bypass, and transaction freezing through flawed monotonicity enforcement and inadequate signature or state validation.  

**Original Text Preview:**

The new implementation of the `ContractDeployer` contract [prevents](https://github.com/matter-labs/era-contracts/blob/cc1619cfb03cc19adb21a2071c89415cab1479e8/system-contracts/contracts/ContractDeployer.sol#L79-L82) accounts from updating their nonce ordering system. Additionally, `KeyedSequential` is specified as the [default ordering](https://github.com/matter-labs/era-contracts/blob/cc1619cfb03cc19adb21a2071c89415cab1479e8/system-contracts/contracts/ContractDeployer.sol#L437-L439), which makes the `Arbitrary` ordering option fully deprecated.

However, there are still places that do not reference this deprecation of the `Arbitrary` ordering which could cause confusion. In particular:

* The [documentation and name](https://github.com/matter-labs/era-contracts/blob/cc1619cfb03cc19adb21a2071c89415cab1479e8/system-contracts/contracts/interfaces/IContractDeployer.sol#L33-L39) of the element in the `enum` corresponding to the `Arbitrary` type in the `IContractDeployer` interface.
* The [broad documentation](https://github.com/matter-labs/era-contracts/blob/cc1619cfb03cc19adb21a2071c89415cab1479e8/docs/l2_system_contracts/system_contracts_bootloader_description.md#L227-L235) of the contract when referencing the nonce ordering.

Consider updating the documentation and `enum` value to properly reflect that this nonce ordering system is now deprecated in order to improve code readability, avoid confusion, and make the current design choice explicit. Additionally, even if there is not any account with `Arbitrary` ordering configured, there is a chance that one could set it before this code is deployed. Consider adding a function that would allow any account configured to use `Arbitrary` ordering to strictly migrate to `KeyedSequential`. This would provide these accounts with a way to correctly migrate in case they unknowingly update to `Arbitrary` before the update.

***Update:** Resolved in [pull request #1387](https://github.com/matter-labs/era-contracts/pull/1387) at commit [051b360](https://github.com/matter-labs/era-contracts/pull/1387/commits/051b360aa8180208d66a8ceb6fbcc49798f8c3dd). The Matter Labs team stated:*

> *Migrating from `Arbitrary` ordering back to `KeyedSequential` is forbidden due to the assumptions that `KeyedSequential` ordering makes. Namely, if nonce value for nonce key K is V, the assumption is that none of the values above V are used. This assumption would break if account migrates from `Arbitrary` ordering.*

---
