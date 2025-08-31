# Cluster -1096

**Rank:** #128  
**Count:** 113  

## Label
Missing authorization and account validation allow attackers to bypass access controls, manipulate state, or execute unauthorized operations through improper instruction checks or flawed account handling.

## Cluster Information
- **Total Findings:** 113

## Examples

### Example 1

**Auto Label:** Inadequate operator authorization validation leads to unauthorized access, inconsistent state, and potential denial-of-service through flawed access controls and missing pre-checks.  

**Original Text Preview:**

## Severity

Low Risk

## Description

The `isAuthorizedOperator()` function in `CredifiERC20Adaptor` implements a custom check for operator authorization, duplicating logic that already exists in the Ethereum Vault Connector (EVC).

## Location of Affected Code

File: [src/CredifiERC20Adaptor.sol#L448](https://github.com/credifi/contracts-audit/blob/ba976bad4afaf2dc068ca9dcd78b38052d3686e3/src/CredifiERC20Adaptor.sol#L448)

```solidity
function isAuthorizedOperator(address user) public view returns (bool) {
   bytes19 addressPrefix = evc.getAddressPrefix(user);

   // Cast to extended interface to access bit field methods
   IEVCExtended evcExtended = IEVCExtended(address(evc));
   uint256 operatorBitField = evcExtended.getOperator(addressPrefix, address(this));

   // Calculate the account ID (bit position) for this user
   // Convert address prefix to owner address using the EVC pattern
   address owner = address(uint160(uint152(addressPrefix)) << 8);
   uint256 accountId = uint160(user) ^ uint160(owner);

   // Check if the bit for this account is set in the operator bit field
   return (operatorBitField & (1 << accountId)) != 0;
}
```

## Recommendation

Consider removing the custom logic:

```solidity
function isAuthorizedOperator(address user) public view returns (bool) {
   return evc.isAccountOperatorAuthorized(user, address(this));
}
```

## Team Response

Fixed.

---
### Example 2

**Auto Label:** Insufficient state validation leads to persistent or incorrect access control, enabling unauthorized interactions, asset draining, or invalid operations.  

**Original Text Preview:**

In `ZipperFactoryVault:createVault`, two mapping values, `tokenToVault` and `vaultToToken`, are assigned. The issue arises when a changer updates these values later using a new vault address. However, the data for the `oldVault` is not deleted, leading to potential issues with outdated mappings.

---
### Example 3

**Auto Label:** Insufficient state validation leads to persistent or incorrect access control, enabling unauthorized interactions, asset draining, or invalid operations.  

**Original Text Preview:**

Each vault must be unique to a token. However, the `createVault` and `changeVault` functions do not check whether the vault key is already assigned to another token. This could result in a vault being overwritten or mistakenly reassigned, violating the uniqueness constraint.

Mitigation:
Add the following check to both functions to ensure the vault is not already in use by another token:

```solidity
require(vaultToToken[chainId][vault].length == 0, "Vault already exists for another token");
```

---
