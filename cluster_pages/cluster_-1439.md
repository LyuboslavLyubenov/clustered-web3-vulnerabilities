# Cluster -1439

**Rank:** #386  
**Count:** 14  

## Label
Inconsistent or inaccurate cap enforcement in withdrawal and redemption limits, leading to user misalignment, fund locking, and potential financial loss due to failed validation of actual availability versus displayed limits.

## Cluster Information
- **Total Findings:** 14

## Examples

### Example 1

**Auto Label:** Inconsistent or inaccurate cap enforcement in withdrawal and redemption limits, leading to user misalignment, fund locking, and potential financial loss due to failed validation of actual availability versus displayed limits.  

**Original Text Preview:**

**Description:** [EIP-4626](https://eips.ethereum.org/EIPS/eip-4626) states on `maxRedeem`:
> MUST factor in both global and user-specific limits, like if redemption is entirely disabled (even temporarily) it MUST return 0.

`pUSDeVault::maxRedeem` doesn't account for redemption pausing, in violation of EIP-4626 which can break protocols integrating with `pUSDeVault`. `MetaVault::redeem` uses `_withdraw` so redemptions will be paused when withdrawals are paused.

**Proof of Concept:**
```solidity
function test_maxRedeem_WhenWithdrawalsPaused() external {
    // user1 deposits $1000 USDe into the main vault
    uint256 user1AmountInMainVault = 1000e18;
    USDe.mint(user1, user1AmountInMainVault);

    vm.startPrank(user1);
    USDe.approve(address(pUSDe), user1AmountInMainVault);
    uint256 user1MainVaultShares = pUSDe.deposit(user1AmountInMainVault, user1);
    vm.stopPrank();

    // admin pauses withdrawals
    pUSDe.setWithdrawalsEnabled(false);

    // doesn't revert but it should since `MetaVault::redeem` uses `_withdraw`
    // and withdraws are paused, so `maxRedeem` should return 0
    assertEq(pUSDe.maxRedeem(user1), user1AmountInMainVault);

    // reverts with WithdrawalsDisabled
    vm.prank(user1);
    pUSDe.redeem(user1MainVaultShares, user1, user1);

    // https://eips.ethereum.org/EIPS/eip-4626 maxRedeem says:
    // MUST factor in both global and user-specific limits,
    // like if redemption are entirely disabled (even temporarily) it MUST return 0
}
```

**Recommended Mitigation:** When withdrawals are paused, `maxRedeem` should return 0. The override of `maxRedeem` should likely be done in `PreDepositVault` because there is where the pausing is implemented.

**Strata:** Fixed in commit [8021069](https://github.com/Strata-Money/contracts/commit/80210696f5ebe73ad7fca071c1c1b7d82e2b02ae).

**Cyfrin:** Verified.

---
### Example 2

**Auto Label:** Inconsistent or inaccurate cap enforcement in withdrawal and redemption limits, leading to user misalignment, fund locking, and potential financial loss due to failed validation of actual availability versus displayed limits.  

**Original Text Preview:**

## Security Review

## Severity
**Low Risk**

## Context
(No context files were provided by the reviewer)

## Description
The `setMinRedemptionAmount` function in `RedeemController` allows setting a minimum redemption amount without an upper limit. While this can be useful for preventing spam or griefing attacks with small redemption requests, an excessively high minimum redemption amount could prevent users from redeeming their funds entirely if their balance is below the set threshold. If `minRedemptionAmount` is set too high, users with smaller balances may be effectively locked out of redemption, which could lead to undesirable user experience and accessibility issues.

## Recommendation
Consider enforcing a reasonable maximum cap on `minRedemptionAmount` to prevent unintended restrictions on redemptions. This can be achieved by adding a validation check in `setMinRedemptionAmount`, such as:

```solidity
require(_minRedemptionAmount <= MAX_REDEMPTION_AMOUNT, "Redemption amount exceeds allowed limit");
```

where `MAX_REDEMPTION_AMOUNT` is defined as a safe upper bound.

## Additional Notes
**infiniFi**: Not implementing this as this action is controlled by Governor role.  
**Spearbit**: Acknowledged by infiniFi team.

---
### Example 3

**Auto Label:** **Incorrect loss validation during withdrawals leads to fund loss or blocked exits, undermining user access and capital preservation under adverse conditions.**  

**Original Text Preview:**

<https://github.com/code-423n4/2024-12-bakerfi/blob/0daf8a0547b6245faed5b6cd3f5daf44d2ea7c9a/contracts/core/MultiStrategy.sol# L148>

<https://github.com/code-423n4/2024-12-bakerfi/blob/0daf8a0547b6245faed5b6cd3f5daf44d2ea7c9a/contracts/core/MultiStrategy.sol# L173>

<https://github.com/code-423n4/2024-12-bakerfi/blob/0daf8a0547b6245faed5b6cd3f5daf44d2ea7c9a/contracts/core/MultiStrategy.sol# L226>

<https://github.com/code-423n4/2024-12-bakerfi/blob/0daf8a0547b6245faed5b6cd3f5daf44d2ea7c9a/contracts/core/MultiStrategy.sol# L264>

### Finding description and impact

When even one third party integrated with MultiStrategyVault is paused, withdrawals become impossible from all Strategies. There is no way to remove the paused third party (Strategy).

### Proof of Concept

Third parties integrated with this project can be paused according to their circumstances. For example, when the AAVE V3 pool is paused, transactions attempting deposits or withdrawals from this pool will revert ([reference](https://github.com/aave/aave-v3-core/blob/782f51917056a53a2c228701058a6c3fb233684a/contracts/protocol/libraries/logic/ValidationLogic.sol# L66-L107)). The [audit scope](https://github.com/code-423n4/2024-12-bakerfi/tree/0daf8a0547b6245faed5b6cd3f5daf44d2ea7c9a?tab=readme-ov-file# external-integrations-eg-uniswap-behavior-in-scope) explicitly states that third party pausability is included.

While it makes sense for a single strategy vault to be paused when an integrated third party is paused, a multi strategy vault should not halt operations with other third parties just because one is paused. This is because funds invested in other strategies could be put at risk due to a single third party.

Therefore, there needs to be a way to temporarily exclude paused third parties (Strategies). Without this ability, transactions will revert because it attempts deposits/withdrawals from the paused third party.

* For deposits, the weight of the strategy linked to that third party can be temporarily set to 0 to exclude it from deposit targets.
* However, for withdrawals, it attempts to withdraw based on the percentage of `totalAssets` deposited, not weight, so even if weight is 0, it attempts to withdraw to the paused third party. This transaction will be reverted, so the user will not be able to withdraw.
* Since withdrawals are impossible, asset reallocation through `rebalance` is also impossible.
* Attempting to remove the strategy connected to that third party using `removeStrategy` will try to withdraw asset tokens for redistribution to other strategies. This will revert, making it impossible to remove the problematic strategy.

In other words, while a paused third party can be excluded from deposits, it cannot be excluded from withdrawals. If that third party’s pool becomes permanently paused, all tokens deposited in the `MultiStrategyVault` will be permanently locked.
```

function _allocateAssets(uint256 amount) internal returns (uint256 totalDeployed) {
    totalDeployed = 0;
    for (uint256 i = 0; i < _strategies.length; ) {
@>      uint256 fractAmount = (amount * _weights[i]) / _totalWeight;
        if (fractAmount > 0) {
@>          totalDeployed += IStrategy(_strategies[i]).deploy(fractAmount);
        }
        unchecked {
            i++;
        }
    }
}

function _deallocateAssets(uint256 amount) internal returns (uint256 totalUndeployed) {
    uint256[] memory currentAssets = new uint256[](_strategies.length);

    uint256 totalAssets = 0;
    uint256 strategiesLength = _strategies.length;

    for (uint256 i = 0; i < strategiesLength; i++) {
@>      currentAssets[i] = IStrategy(_strategies[i]).totalAssets();
        totalAssets += currentAssets[i];
    }
    totalUndeployed = 0;
    for (uint256 i = 0; i < strategiesLength; i++) {
@>      uint256 fractAmount = (amount * currentAssets[i]) / totalAssets;
@>      totalUndeployed += IStrategy(_strategies[i]).undeploy(fractAmount);
    }
}

function _rebalanceStrategies(uint256[] memory indexes, int256[] memory deltas) internal {
    ...
    // Iterate through each strategy to adjust allocations
    for (uint256 i = 0; i < totalStrategies; i++) {
        // if the delta is 0, we don't need to rebalance the strategy
        if (deltas[i] == 0) continue;

        // if the delta is positive, we need to deploy the strategy
        if (deltas[i] > 0) {
            uint256 balanceOf = IERC20(_strategies[indexes[i]].asset()).balanceOf(address(this));
            uint256 amount = uint256(deltas[i]) > balanceOf ? balanceOf : uint256(deltas[i]);
            IStrategy(_strategies[indexes[i]]).deploy(amount);
            // if the delta is negative, we need to undeploy the strategy
        } else if (deltas[i] < 0) {
@>          IStrategy(_strategies[indexes[i]]).undeploy(uint256(-deltas[i]));
        }
    }
    ...
}

function removeStrategy(uint256 index) external onlyRole(VAULT_MANAGER_ROLE) {
    ...
    // If the strategy has assets, undeploy them and allocate accordingly
@>  if (strategyAssets > 0) {
@>      IStrategy(_strategies[index]).undeploy(strategyAssets);
        _allocateAssets(strategyAssets);
    }
    ...
}
```

### Recommended Mitigation Steps

We need a way to exclude third parties (Strategies) from withdrawals if they are unavailable. We need to be able to exclude a strategy without making a withdrawal request.

**chefkenji (BakerFi) acknowledged**

---

---
