# Cluster -1228

**Rank:** #361  
**Count:** 16  

## Label
Inadequate validation logic in user operation processing leads to incorrect error handling, signature reuse, policy bypass, and inconsistent state checks, enabling replay attacks, validation failures, and compromised security.

## Cluster Information
- **Total Findings:** 16

## Examples

### Example 1

**Auto Label:** Inadequate validation logic in user operation processing leads to incorrect error handling, signature reuse, policy bypass, and inconsistent state checks, enabling replay attacks, validation failures, and compromised security.  

**Original Text Preview:**

## Context
(No context files were provided by the reviewer)

## Description
In the SmartSession codebase, at least one action policy is generally required to be set and verified for `validateUserOp()` to succeed. This is enforced with a `minPolicies` argument of 1, and if this threshold isn't met initially, the system re-attempts validation using the `FALLBACK_ACTIONID`.

However, there is currently one scenario where `validateUserOp()` can succeed without validating any action policies. Specifically, in the `checkBatch7579Exec()` function, it is technically possible for the `executions` array to be empty in the following code:

```solidity
function checkBatch7579Exec(
    mapping(ActionId => Policy) storage $policies,
    PackedUserOperation calldata userOp,
    PermissionId permissionId,
    uint256 minPolicies
)
    internal
    returns (ValidationData vd)
{
    // Decode the batch of 7579 executions from the user operation's call data
    Execution[] calldata executions = userOp.callData.decodeUserOpCallData().decodeBatch();
    uint256 length = executions.length;
    
    // Iterate through each execution in the batch
    for (uint256 i; i < length; i++) {
        Execution calldata execution = executions[i];
        // Check policies for the current execution and intersect the result with previous checks
        ValidationData _vd = checkSingle7579Exec({
            $policies: $policies,
            userOp: userOp,
            permissionId: permissionId,
            target: execution.target,
            value: execution.value,
            callData: execution.callData,
            minPolicies: minPolicies
        });
        vd = vd.intersect(_vd);
    }
}
```

If the `executions` array is empty, no action policies are checked, since no calls are ever made. While this may not be directly exploitable, it introduces a potential deviation from the codebase's intended design. Also, even though this would result in a no-op call to the account's `execute()` function, gas fees may still be incurred by the account, so it's important to ensure the intended validation is always enforced.

## Recommendation
To prevent this behavior, consider enforcing that the `executions` array is non-empty in `checkBatch7579Exec()`.

## Rhinestone
Fixed in PR 106.

## Cantina Managed
Verified.

---
### Example 2

**Auto Label:** Inadequate validation logic in user operation processing leads to incorrect error handling, signature reuse, policy bypass, and inconsistent state checks, enabling replay attacks, validation failures, and compromised security.  

**Original Text Preview:**

## Validator User Operation Review

## Context
(No context files were provided by the reviewer)

## Description
In the `validateUserOp()` function, the `_enablePolicies()` logic allows an account to atomically install policies during validation. Part of the data provided to the installation logic is `enableData.sessionToEnable.erc7739Policies.allowedERC7739Content`, which is an array of type names that are allowed to be used during an ERC-7739 compliant `isValidSignature()` call.

Despite this data being provided to `_enablePolicies()`, there is currently no logic to use it. As a result, accounts that install policies through `_enablePolicies()` will have incorrectly installed permissions, and future calls to `isValidSignature()` will fail.

## Recommendation
Add logic to `_enablePolicies()` to install `enableData.sessionToEnable.erc7739Policies.allowedERC7739Content`:

```php
// Enable ERC1271 policies
$erc1271Policies.enable({
    policyType: PolicyType.ERC1271,
    permissionId: permissionId,
    configId: permissionId.toErc1271PolicyId().toConfigId(),
    policyDatas: enableData.sessionToEnable.erc7739Policies.erc1271Policies,
    smartAccount: account,
    useRegistry: useRegistry
});
$enabledERC7739Content.enable(enableData.sessionToEnable.erc7739Policies.allowedERC7739Content, permissionId, account);
```

## Rhinestone
Fixed in PR 71.

## Cantina Managed
Verified.

---
### Example 3

**Auto Label:** Failure to validate full user operation structure and context leads to signature forgery, session key hijacking, and unauthorized transaction execution.  

**Original Text Preview:**

## Medium Risk Report

**Severity:** Medium Risk  
**Context:** AtlasVerification.sol#L532-L575  

**Description:**  
The function `_verifyUser()` uses the ERC4337 function `validateUserOp()` to validate smart contract wallets. There are several reasons why this won't work:

- `entryPoint v0.6` has a different layout for `UserOperation`.
- `entryPoint v0.7` has yet another layout for `UserOperation`.
- Smart wallets usually allow only calls from the `EntryPoint` to `validateUserOp`, as seen in `BaseAccount.sol`.
- Any random smart contract that has a fallback function that returns `0` on unknown functions would satisfy this check.

**Note:** Other ERC-4337 wallets usually don't put a gas limit when calling `validateUserOp()`.

```solidity
function _verifyUser( /*...*/ ) /*...*/ {
    if (userOp.from.code.length > 0) {
        // ...
        bool validSmartWallet =
        IAccount(userOp.from).validateUserOp{ gas: 30_000 }(userOp, _getProofHash(userOp), 0) == 0;
        return (isSimulation || validSmartWallet);
    }
    // ...
}
```

**Recommendation:**  
`ERC1271.isValidSignature()` seems a more logical solution. Also see OZ `SignatureChecker`. However, be aware of implementation issues of `ERC1271.isValidSignature()`.

**Fastlane:** Solved by PR 250.  
**Spearbit:** Verified.

---
