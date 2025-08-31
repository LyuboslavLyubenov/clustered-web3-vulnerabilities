# Cluster -1025

**Rank:** #313  
**Count:** 22  

## Label
Failure to increment critical state nonces during operator operations, leading to inconsistent validator key indexing, compromised signature validation, and potential nonce-based attacks or state corruption.

## Cluster Information
- **Total Findings:** 22

## Examples

### Example 1

**Auto Label:** Failure to increment critical state nonces during operator operations, leading to inconsistent validator key indexing, compromised signature validation, and potential nonce-based attacks or state corruption.  

**Original Text Preview:**

## Security Assessment Report

## Severity
**Low Risk**

## Context
`StakingManager.sol#L548-L575`

## Description
In the `StakingManager` contract, the `queueL1Operations` function allows a trusted role (`OPERATOR_ROLE`) to enqueue a list of validator-related operations. However, it does not validate that the provided `validators[i]` addresses are legitimate or currently active validators. This contrasts with other functions in the contract that explicitly check validator status using `ValidatorManager`, e.g., via `validatorActiveState()` or `getDelegation()`.

While `queueL1Operations` is restricted to operator-only access, the absence of a validation step leaves room for accidental queuing of invalid or inactive validator addresses. This may result in wasted queue entries, failed execution attempts, or the need for manual correction/reset of the queue.

## Recommendation
Consider validating each validator address against `ValidatorManager` during `queueL1Operations`, such as by checking:

```solidity
require(validatorManager.validatorActiveState(validators[i]), "Invalid validator");
```

This would bring consistency with other validator-related operations and help reduce the potential for operational mistakes, even under trusted roles.

## Fixes
- **Kinetiq:** Fixed in commit `e2bac5c`.
- **Cantina Managed:** Fixed in commit `e2bac5c` by implementing the validator check. Moreover, the Kinetiq team added a flow control to perform the check just for deposit operations, thus making the function robust, consistent, and secure.

---
### Example 2

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
### Example 3

**Auto Label:** **Improper nonce handling leading to value collisions, overflow, or incorrect validation, compromising order uniqueness and system integrity.**  

**Original Text Preview:**

##### Description
The issue is identified within the [`LOB`](https://github.com/longgammalabs/hanji-contracts/blob/09b6188e028650b9c1758010846080c5f8c80f8e/src/OnchainLOB.sol#L57) contract.

The initialization of the `nonce` variable is incorrect. The current implementation `uint64(1) << nonce_length - uint64(1)` sets it to `2^38` instead of `2^39 - 1`.

##### Recommendation
We recommend initializing `nonce` as `(1 << nonce_length) - 1` to set it correctly to `2^39 - 1`.

---
