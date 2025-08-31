# Cluster -1254

**Rank:** #364  
**Count:** 16  

## Label
Failure to enforce critical invariants through proper validation and state checks, leading to incorrect state transitions, loss of error context, or unintended behavior under specific conditions.

## Cluster Information
- **Total Findings:** 16

## Examples

### Example 1

**Auto Label:** Failure to enforce state invariants during critical operations, enabling unauthorized state transitions, disruption of recovery processes, or persistent stale data exposure.  

**Original Text Preview:**

The [`onUninstall` function](https://github.com/matter-labs/zksync-sso-clave-contracts/blob/c7714c0fe0a33a23acce5aa20355f088d330b4f7/src/validators/GuardianRecoveryValidator.sol#L84) can clear all data associated with guardians of a specific account, but it cannot clear the `pendingRecoveryData` mapping's requests. This omission allows an account to re-connect the `GuardianRecoveryValidator` contract and execute pending recovery requests within the permitted time window. This behavior contradicts the intended logic, whereby a newly reconnected account should not be able to perform actions such as calling the `addValidationKey` function from the `WebAuthValidator` contract without waiting for the required delay, as the state is expected to be cleared.

Consider explicitly clearing the `pendingRecoveryData` mapping's requests during the `onUninstall` function to prevent unauthorized immediate account recovery based on the old state of the `pendingRecoveryData` mapping.

***Update:** Resolved in [pull request #342](https://github.com/matter-labs/zksync-sso-clave-contracts/pull/342/files) at commit [69f4c36](https://github.com/matter-labs/zksync-sso-clave-contracts/pull/342/commits/69f4c36abd76f6cb8cdf481ad341d1417ed99d69).*

---
### Example 2

**Auto Label:** Failure to enforce critical invariants through proper validation and state checks, leading to incorrect state transitions, loss of error context, or unintended behavior under specific conditions.  

**Original Text Preview:**

The event `MessageCached` on `Beacon.sol#_lzReceive()` will get emitted in case the execute of `safeCall()` failed

```solidity
            emit MessageCached(
                _origin.srcEid,
                _origin.sender,
                _origin.nonce,
                reason
            );
```

However, by setting zero as the `maxCopy` value in `safeCall()` the `reason` will always be empty.
Consider cache at least the few first bytes will help.

---
### Example 3

**Auto Label:** Inconsistent state management and poor validation mechanisms lead to data corruption, unintended state transitions, and compromised system integrity during upgrades or migrations.  

**Original Text Preview:**

## Code Quality Improvements

**Context:** (No context files were provided by the reviewer)

Following are some suggested code quality improvements:

### Suggested Improvements

1. **Sethunger of new wolf directly to `ONEashunger[wolfID]` of a new wolf will always be 0:**
   - `WolfNFT.sol#L61`.

2. **Use constants where possible:**
   - `ONE_WEEK` in `SheepV3.sol#L397`.
   - `mintPrice`, `teamCut` and `maxPreMint` in `SheepV3.sol#L400-L402`.

3. **Use immutables where possible:**
   - `ERC20` name and symbol in `SheepV3.sol#L58-L59`.

4. **Remove unused storage variables:**
   - `sheepToClaim` in `sheepDog.sol#L2`.

5. **Remove unused events:**
   - `sheepBorn`, `sheepSlaughtered`, `lassieReleased`, `sheepPastured` and `newSheppard` in `SheepV3.sol#L413-L415`.
   - `newSheppard` in `SheepV3.sol#L417`.

6. **Remove `beforeTokenTransfer` and `afterTokenTransfer` as they are not implemented:**
   - `SheepV3.sol#L239`.
   - `SheepV3.sol#L252`.
   - `SheepV3.sol#L267`.
   - `SheepV3.sol#L276`.
   - `SheepV3.sol#L293`.
   - `SheepV3.sol#L305`.
   - `SheepV3.sol#L361`.
   - `SheepV3.sol#L377`.

### Additional Notes

- **Ceazor Snack Sandwich:** Fixed in `c46d082`.
- **Cantina:** Fix OK.

---
