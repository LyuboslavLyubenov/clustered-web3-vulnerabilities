# Cluster -1430

**Rank:** #416  
**Count:** 11  

## Label
Misuse of comparison operators in deposit caps leads to incorrect limits, allowing partial or failed deposits, and violates ERC-4626 compliance by failing to enforce maximum deposit bounds under all conditions.

## Cluster Information
- **Total Findings:** 11

## Examples

### Example 1

**Auto Label:** Failure to return zero in maxDeposit/maxMint when operations are paused, violating EIP-4626 by enabling incorrect assumptions and inconsistent behavior under disabled states.  

**Original Text Preview:**

**Description:** [EIP-4626](https://eips.ethereum.org/EIPS/eip-4626) states on `maxMint`:
> MUST factor in both global and user-specific limits, like if mints are entirely disabled (even temporarily) it MUST return 0.

`pUSDeVault::maxMint` doesn't account for mint pausing, in violation of EIP-4626 which can break protocols integrating with `pUSDeVault`. Since `MetaVault::mint` uses `_deposit`, mints will be paused when deposits are paused.

**Proof of Concept:**
```solidity
function test_maxMint_WhenDepositsPaused() external {
    // admin pauses deposists
    pUSDe.setDepositsEnabled(false);

    // should revert here as maxMint should return 0
    // since deposits are paused and `MetaVault::mint` uses `_deposit`
    assertEq(pUSDe.maxMint(user1), type(uint256).max);

    // attempt to mint to show the error
    uint256 user1AmountInMainVault = 1000e18;
    USDe.mint(user1, user1AmountInMainVault);

    vm.startPrank(user1);
    USDe.approve(address(pUSDe), user1AmountInMainVault);
    // reverts with DepositsDisabled since `MetaVault::mint` uses `_deposit`
    uint256 user1MainVaultShares = pUSDe.mint(user1AmountInMainVault, user1);
    vm.stopPrank();

    // https://eips.ethereum.org/EIPS/eip-4626 maxMint says:
    // MUST factor in both global and user-specific limits,
    // like if mints are entirely disabled (even temporarily) it MUST return 0.
}
```

**Recommended Mitigation:** When deposits are paused, `maxMint` should return 0. The override of `maxMint` should likely be done in `PreDepositVault` because there is where the pausing is implemented.

**Strata:** Fixed in commit [8021069](https://github.com/Strata-Money/contracts/commit/80210696f5ebe73ad7fca071c1c1b7d82e2b02ae).

**Cyfrin:** Verified.

---
### Example 2

**Auto Label:** Misuse of comparison operators in deposit caps leads to incorrect limits, allowing partial or failed deposits, and violates ERC-4626 compliance by failing to enforce maximum deposit bounds under all conditions.  

**Original Text Preview:**

**Description:** [EIP-4626](https://eips.ethereum.org/EIPS/eip-4626) states on `maxDeposit`:
> MUST factor in both global and user-specific limits, like if deposits are entirely disabled (even temporarily) it MUST return 0.

`pUSDeVault::maxDeposit` doesn't account for deposit pausing, in violation of EIP-4626 which can break protocols integrating with `pUSDeVault`.

**Proof of Concept:**
```solidity
function test_maxDeposit_WhenDepositsPaused() external {
    // admin pauses deposists
    pUSDe.setDepositsEnabled(false);

    // reverts as maxDeposit returns uint256.max even though
    // attempting to deposit would revert
    assertEq(pUSDe.maxDeposit(user1), 0);

    // https://eips.ethereum.org/EIPS/eip-4626 maxDeposit says:
    // MUST factor in both global and user-specific limits,
    // like if deposits are entirely disabled (even temporarily) it MUST return 0.
}
```

**Recommended Mitigation:** When deposits are paused, `maxDeposit` should return 0. The override of `maxDeposit` should likely be done in `PreDepositVault` because there is where the pausing is implemented.

**Strata:** Fixed in commit [8021069](https://github.com/Strata-Money/contracts/commit/80210696f5ebe73ad7fca071c1c1b7d82e2b02ae).

**Cyfrin:** Verified.

---
### Example 3

**Auto Label:** Misuse of comparison operators in deposit caps leads to incorrect limits, allowing partial or failed deposits, and violates ERC-4626 compliance by failing to enforce maximum deposit bounds under all conditions.  

**Original Text Preview:**

##### Description
The issue is identified within the function [`depositERC20`](https://github.com/LayerLabs-app/Ozean-Contracts/blob/dacd9ed9c895c9b6be422531eea15fecc3c2b1d8/src/L1/LGEStaking.sol#L106) of contract `LGEStaking`. Currently, the deposit requirement uses a strict `<` comparison when validating the deposit against the cap, whereas `<=` should be used to correctly enforce the maximum deposit limit. The same issue is present in multiple places: [`depositETH` function](https://github.com/LayerLabs-app/Ozean-Contracts/blob/dacd9ed9c895c9b6be422531eea15fecc3c2b1d8/src/L1/LGEStaking.sol#L128), [`bridge` function](https://github.com/LayerLabs-app/Ozean-Contracts/blob/dacd9ed9c895c9b6be422531eea15fecc3c2b1d8/src/L1/USDXBridge.sol#L115).

This is classified as **Low** severity, because while it does not entirely break contract logic, it may not allow to fully use-up all the deposit cap.

##### Recommendation
We recommend changing the comparison from `<` to `<=` to ensure that deposits can reach the defined cap.

---
