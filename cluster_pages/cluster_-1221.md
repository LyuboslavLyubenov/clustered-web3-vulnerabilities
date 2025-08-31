# Cluster -1221

**Rank:** #117  
**Count:** 139  

## Label
Failure to validate critical state conditions allows attackers to bypass access controls, execute invalid operations, or manipulate system state, leading to unauthorized transactions, denial of service, or inconsistent outcomes.

## Cluster Information
- **Total Findings:** 139

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

**Auto Label:** Insufficient state validation leads to persistent or incorrect access control, enabling unauthorized interactions, asset draining, or invalid operations.  

**Original Text Preview:**

**Description:** When supporting additional vaults, `MetaVault::addVault` should enforce that the new vault being supported has an identical underlying base asset as itself. Otherwise:
* `redeemRequiredBaseAssets` won't work as expected since the newly supported vault doesn't have the same base asset
* `MetaVault::depositedBase` will become corrupt, especially if the underlying asset tokens use different decimal precision

**Proof of Concept:**
```solidity
function test_vaultSupportedWithDifferentUnderlyingAsset() external {
    // create ERC4626 vault with different underlying ERC20 asset
    MockUSDe differentERC20 = new MockUSDe();
    MockERC4626 newSupportedVault = new MockERC4626(differentERC20);

    // verify pUSDe doesn't have same underlying asset as new vault
    assertNotEq(pUSDe.asset(), newSupportedVault.asset());

    // but still allows it to be added
    pUSDe.addVault(address(newSupportedVault));

    // this breaks `MetaVault::redeemRequiredBaseAssets` since
    // the newly supported vault doesn't have the same base asset
}
```

**Recommended Mitigation:** Change `MetaVault::addVaultInner`:
```diff
    function addVaultInner (address vaultAddress) internal {
+       IERC4626 newVault = IERC4626(vaultAddress);
+       require(newVault.asset() == asset(), "Vault asset mismatch");
```

**Strata:** Fixed in commits [9e64f09](https://github.com/Strata-Money/contracts/commit/9e64f09af6eb927c9c736796aeb92333dbb72c18), [706c2df](https://github.com/Strata-Money/contracts/commit/706c2df3f2caf6651b1d8e858beb5097dbd7d066).

**Cyfrin:** Verified.

---
