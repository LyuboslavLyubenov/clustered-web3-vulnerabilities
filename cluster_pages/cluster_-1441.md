# Cluster -1441

**Rank:** #80  
**Count:** 212  

## Label
Misinterpretation of asset-sharing semantics and parameter logic leads to incorrect fund withdrawals, fee miscalculations, and inconsistent limits, causing permanent user losses or protocol incompatibility.

## Cluster Information
- **Total Findings:** 212

## Examples

### Example 1

**Auto Label:** Misinterpretation of asset-sharing semantics and parameter logic leads to incorrect fund withdrawals, fee miscalculations, and inconsistent limits, causing permanent user losses or protocol incompatibility.  

**Original Text Preview:**

The `_deposit` and `_withdraw` functions are limited by `depositCap` and `idleBalance` respectively, but these limitations are not implemented in the ERC4626 functions `maxDeposit` and `maxWithdraw`.

This could cause integration compatibility issues with other protocols that expect a compliant vault, particularly if they rely on `maxDeposit` and `maxWithdraw` to determine the permissible deposit/withdrawal amounts.

Impact: Indirect loss of protocol fees if other protocols cannot integrate with the contract, or locked funds in some edge cases.

Likelihood: While using a wrapper could avoid this issue, it might be impossible with some protocols that expect a compliant vault.

Recommendations:

Consider overriding `maxDeposit` and `maxWithdraw` to enforce the same limits used in `_deposit` and `_withdraw`, ensuring the contract properly reports the depositable/withdrawable amounts.

---
### Example 2

**Auto Label:** Misinterpretation of asset-sharing semantics and parameter logic leads to incorrect fund withdrawals, fee miscalculations, and inconsistent limits, causing permanent user losses or protocol incompatibility.  

**Original Text Preview:**

**Description:** Unlike `MetaVault::deposit`, `MetaVault::mint`, and `MetaVault::withdraw` which all invoke the corresponding `IERC4626` function, `MetaVault::redeem` erroneously calls `ERC4626Upgradeable::withdraw` when attempting to redeem `USDe` from `pUSDeVault`:

```solidity
function redeem(address token, uint256 shares, address receiver, address owner) public virtual returns (uint256) {
    if (token == asset()) {
        return withdraw(shares, receiver, owner);
    }
    ...
}
```

**Impact:** The behavior of `MetaVault::redeem` differs from that which is expected depending on whether `token` is specified as `USDe` or one of the other supported vault tokens.

**Recommended Mitigation:**
```diff
    function redeem(address token, uint256 shares, address receiver, address owner) public virtual returns (uint256) {
        if (token == asset()) {
--          return withdraw(shares, receiver, owner);
++          return redeem(shares, receiver, owner);
        }
        ...
    }
```

**Strata:** Fixed in commit [7665e7f](https://github.com/Strata-Money/contracts/commit/7665e7f3cd44d8a025f555737677d2014f4ac8a8).

**Cyfrin:** Verified.

---
### Example 3

**Auto Label:** Misinterpretation of asset-sharing semantics and parameter logic leads to incorrect fund withdrawals, fee miscalculations, and inconsistent limits, causing permanent user losses or protocol incompatibility.  

**Original Text Preview:**

**Description:** `MetaVault::redeemRequiredBaseAssets` is supposed to iterate through the supported vaults, redeeming assets until the required amount of base assets is obtained:
```solidity
/// @notice Iterates through supported vaults and redeems assets until the required amount of base tokens is obtained
```

Its implementation however only retrieves from a supported vault if that one withdrawal can satisfy the desired amount:
```solidity
function redeemRequiredBaseAssets (uint baseTokens) internal {
    for (uint i = 0; i < assetsArr.length; i++) {
        IERC4626 vault = IERC4626(assetsArr[i].asset);
        uint totalBaseTokens = vault.previewRedeem(vault.balanceOf(address(this)));
        // @audit only withdraw if a single withdraw can satisfy desired amount
        if (totalBaseTokens >= baseTokens) {
            vault.withdraw(baseTokens, address(this), address(this));
            break;
        }
    }
}
```

**Impact:** This has a number of potential problems:
1) if no single withdraw can satisfy the desired amount, then the calling function will revert due to insufficient funds even if the desired amount could be satisfied by multiple smaller withdrawals from different supported vaults
2) a single withdraw may be greater than the desired amount, leaving `USDe` tokens inside the vault contract. This is suboptimal as then they would not be earning yield by being staked in `sUSDe`, and there appears to be no way for the contract owner to trigger the staking once the yield phase has started, since supporting vaults can be added and deposits for them work during the yield phase

**Recommended Mitigation:** `MetaVault::redeemRequiredBaseAssets` should:
* keep track of the total currently redeemed amount
* calculate the remaining requested amount as the requested amount minus the total currently redeemed amount
* if the current vault is not able to redeem the remaining requested amount, redeem as much as possible and increase the total currently redeemed amount by the amount redeemed
* if the current vault could redeem more than the remaining requested amount, redeem only enough to satisfy the remaining requested amount

The above strategy ensures that:
* small amounts from multiple vaults can be used to fulfill the requested amount
* greater amounts than requested are not withdrawn, so no `USDe` tokens remain inside the vault unable to be staked and not earning yield

**Strata:** Fixed in commits [4efba0c](https://github.com/Strata-Money/contracts/commit/4efba0c484a3bd6d4934e0f1ec0eb91848c94298), [7e6e859](https://github.com/Strata-Money/contracts/commit/7e6e8594c05ea7e3837ddbe7395b4a15ea34c7e9).

**Cyfrin:** Verified.

---
