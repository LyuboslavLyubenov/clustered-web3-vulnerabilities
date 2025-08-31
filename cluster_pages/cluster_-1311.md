# Cluster -1311

**Rank:** #105  
**Count:** 161  

## Label
Failure to validate or enforce whitelisted entity constraints leads to unauthorized access, redundant operations, or persistent exposure to malicious actors through inadequate input checks, state management, or control flow design.

## Cluster Information
- **Total Findings:** 161

## Examples

### Example 1

**Auto Label:** Insufficient state validation leads to persistent or incorrect access control, enabling unauthorized interactions, asset draining, or invalid operations.  

**Original Text Preview:**

In `ZipperFactoryVault:createVault`, two mapping values, `tokenToVault` and `vaultToToken`, are assigned. The issue arises when a changer updates these values later using a new vault address. However, the data for the `oldVault` is not deleted, leading to potential issues with outdated mappings.

---
### Example 2

**Auto Label:** Insufficient state validation leads to persistent or incorrect access control, enabling unauthorized interactions, asset draining, or invalid operations.  

**Original Text Preview:**

Each vault must be unique to a token. However, the `createVault` and `changeVault` functions do not check whether the vault key is already assigned to another token. This could result in a vault being overwritten or mistakenly reassigned, violating the uniqueness constraint.

Mitigation:
Add the following check to both functions to ensure the vault is not already in use by another token:

```solidity
require(vaultToToken[chainId][vault].length == 0, "Vault already exists for another token");
```

---
### Example 3

**Auto Label:** Failure to validate or enforce whitelisted entity constraints leads to unauthorized access, redundant operations, or persistent exposure to malicious actors through inadequate input checks, state management, or control flow design.  

**Original Text Preview:**

The `_whitelist` function in the Voter contract allows tokens to be added to a whitelist, but once a token is whitelisted, it cannot be removed. This `one-way` operation may lead to operational inconveniences and potential misuse.

```solidity
function _whitelist(address _token) internal {
    require(!isWhitelisted[_token]);
    isWhitelisted[_token] = true;
    emit Whitelisted(msg.sender, _token);
}
```

Once a token is added to the whitelist, there is no mechanism provided in the contract to remove it from the whitelist. This means that any token that is whitelisted remains so indefinitely, regardless of changes in the token's status or relevance to the system.

To enhance the flexibility and security of the contract, it is recommended to implement a mechanism to allow for the removal of tokens from the whitelist.

---
