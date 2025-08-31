# Cluster -1328

**Rank:** #65  
**Count:** 279  

## Label
Failure to validate array length equality during iteration leads to out-of-bounds access, incorrect state updates, or infinite loops, enabling denial-of-service, data corruption, or unintended transactions.

## Cluster Information
- **Total Findings:** 279

## Examples

### Example 1

**Auto Label:** Failure to validate array length equality during iteration leads to out-of-bounds access, incorrect state updates, or infinite loops, enabling denial-of-service, data corruption, or unintended transactions.  

**Original Text Preview:**

In the [`_executeCall` function](https://github.com/matter-labs/zksync-sso-clave-contracts/blob/c7714c0fe0a33a23acce5aa20355f088d330b4f7/src/SsoAccount.sol#L128) of the `SsoAccount` contract, there is a slice operation performed on the `_data` parameter when handling a `DEPLOYER_SYSTEM_CONTRACT` call. The purpose of this slice operation is to retrieve the function selector, which requires at least 4 bytes of data. However, the current implementation does not validate whether the input data has sufficient length. If the provided data is shorter than 4 bytes, the slicing operation will cause a panic, abruptly terminating the execution.

Consider verifying the length of the `_data` parameter prior to slicing, and explicitly reverting with a meaningful error message if the length is insufficient.

***Update:** Resolved in [pull request #369](https://github.com/matter-labs/zksync-sso-clave-contracts/pull/369) at commit [f476fb1](https://github.com/matter-labs/zksync-sso-clave-contracts/pull/369/commits/f476fb1f064a180ddfcb906c8468dac98c99da1d).*

---
### Example 2

**Auto Label:** Failure to validate array length equality during iteration leads to out-of-bounds access, incorrect state updates, or infinite loops, enabling denial-of-service, data corruption, or unintended transactions.  

**Original Text Preview:**

In the [`addOidcAccount` function](https://github.com/matter-labs/zksync-sso-clave-contracts/blob/ed21d09add8da99d9c82d0f7c30659625c6636e6/src/validators/OidcRecoveryValidator.sol#L163), the returned boolean indicates whether an account has been newly added or merely updated. The function [checks](hhttps://github.com/matter-labs/zksync-sso-clave-contracts/blob/ed21d09add8da99d9c82d0f7c30659625c6636e6/src/validators/OidcRecoveryValidator.sol#L167) the current state by inspecting the length of the `oidcDigest` field within the stored account data, which is then emitted via the `OidcAccountUpdated` event and returned by the function. According to the [documentation](https://github.com/matter-labs/zksync-sso-clave-contracts/blob/ed21d09add8da99d9c82d0f7c30659625c6636e6/src/validators/OidcRecoveryValidator.sol#L162), the function should return `true` if a new key is added and `false` if an existing key is updated.

However, the logic currently implemented misunderstands how length is calculated for the `bytes32` type. The length of a `bytes32` value is always 32 bytes, regardless of the content, meaning the `accountData[msg.sender].oidcDigest.length == 0` condition will consistently evaluate to `false`. Consequently, the returned value and the emitted event will incorrectly indicate that an account is always updated rather than newly created, potentially causing confusion or misinterpretation of account status. It is worth mentioning that the [correct check](https://github.com/matter-labs/zksync-sso-clave-contracts/blob/ed21d09add8da99d9c82d0f7c30659625c6636e6/src/validators/OidcRecoveryValidator.sol#L319) is being performed in the `oidcDataForAddress` function.

In favor of being able to detect when a new key is added and improving code consistency, consider whether comparing the value of `oidcDigest` to zero (e.g., `oidcDigest == bytes32(0)`) might more accurately reflect whether the account is new, rather than relying on its length.

***Update:** Resolved in [pull request #424](https://github.com/matter-labs/zksync-sso-clave-contracts/pull/424) at commit [d7a71ca](https://github.com/matter-labs/zksync-sso-clave-contracts/pull/424/commits/d7a71caed03e2163afe3ff98c9144e529fb530b3). The Matter Labs team stated:*

> *This issue has no PR for itself because it was resolved as part of `L-07`.*

---
### Example 3

**Auto Label:** Failure to validate array length equality during iteration leads to out-of-bounds access, incorrect state updates, or infinite loops, enabling denial-of-service, data corruption, or unintended transactions.  

**Original Text Preview:**

The arrays in `depositSums`, `collateralGainsByDepositor`, `epochToScaleToSums`, and `lastCollateralError_Offset` have a size of 256. However, in the `enableCollateral` function there is no check for the total number of collaterals already enabled in the contract.

If more than 256 collaterals are enabled, the operations that used the arrays mentioned above will revert due to an out-of-bounds access.

Consider adding a check for the total number of collaterals enabled in the `enableCollateral` function.

```diff
+           require(length < 256);
            collateralTokens.push(_collateral);
            indexByCollateral[_collateral] = collateralTokens.length;
```

---
