# Cluster -1069

**Rank:** #235  
**Count:** 41  

## Label
Inadequate slashing penalty calculation and distribution due to flawed balance scaling and improper share tracking, enabling stakers to evade penalties and misallocate penalties across cohorts.

## Cluster Information
- **Total Findings:** 41

## Examples

### Example 1

**Auto Label:** Inconsistent emission indexing and reward calculations due to stale or missing updates to reward trackers, leading to incorrect allocations, overpayment, and broken reward continuity.  

**Original Text Preview:**

## Vulnerability Report

## Summary
The vulnerability concerns the potential for an emission to be removed from SyMeta in a way that breaks the integrity of the system. 

## Details
The **Position** structure tracks the state of a user’s position, including their amount and a list of rewards, which are tied to emissions. Each **Reward** in **Position** corresponds to an **Emission** on SyMeta, tracked by its *mint* and *last_seen_share_index*, which is saved in **Reward**. 

The `Position::ensure_trackers` function ensures that the position has a corresponding reward entry for every emission that exists.

### Code Snippet
```rust
pub fn ensure_trackers(&mut self, emissions: &Vec<Emission>) {
    for (index, emission) in emissions.iter().enumerate() {
        match self.reward_indexes.get_mut(index) {
            Some(reward) => {
                assert_eq!(
                    reward.mint, emission.mint,
                    "Reward mint does not match emission tracker index"
                );
                // [...]
            }
        }
    }
}
```

The **RemoveEmission** operation enables removing an emission from SyMeta, shifting the remaining emissions. In the context of `Position::ensure_trackers`, if an emission is removed from the list, then all subsequent emissions will shift by one index. As the mint is already saved on all the user positions’ rewards, the reward mint will not match the corresponding emission tracker index, causing the function to fail. 

The exponent-core relies on the order of rewards in the position. If emissions are removed and the order is disrupted, the protocol will no longer safely calculate rewards or track their distribution, disrupting the entire protocol and resulting in a denial-of-service for all positions.

## Remediation
- Disallow the **RemoveEmission** operation.

## Patch
- Resolved in **#1914**.

---
### Example 2

**Auto Label:** Inadequate slashing penalty calculation and distribution due to flawed balance scaling and improper share tracking, enabling stakers to evade penalties and misallocate penalties across cohorts.  

**Original Text Preview:**

## Description

Due to rounding, it is possible for the `slashingFactor` to be rounded down to zero, even when both `operatorMaxMagnitude` and `beaconChainSlashingFactor` are non-zero. This can result in incorrect `withdrawableShares` being calculated for the `beaconChainETHStrategy`.

The `slashingFactor` for the `beaconChainETHStrategy` is calculated as:

```solidity
DelegationManager.sol::_getSlashingFactor()
if (strategy == beaconChainETHStrategy) {
    uint64 beaconChainSlashingFactor = eigenPodManager.beaconChainSlashingFactor(staker);
    return operatorMaxMagnitude.mulWad(beaconChainSlashingFactor);
}
```

Due to the rounding of the `mulWad()` function, it is possible for the `slashingFactor` to be rounded down to 0, even when both `operatorMaxMagnitude` and `beaconChainSlashingFactor` are non-zero.

An operator can exploit this rounding to burn their stakers’ withdrawable shares by doing the following:

1. The operator creates a malicious AVS and registers and allocates to it.
2. The operator slashes themself through their AVS by a large amount such that their `maxMagnitude` is reduced to a very small value (e.g. 1).
3. A staker verifies a validator on their EigenPod and delegates to the operator.
4. The staker gets penalised on the beacon chain by a very small amount (e.g. 1 gwei).

Once the staker’s `beaconChainSlashingFactor` is reduced below WAD in step 4, their `slashingFactor` becomes 0 and they lose all their withdrawable shares.

Though the impact of this issue can be extremely severe, this issue has an informational severity rating as it is very unlikely that an operator would grief their stakers without any potential upside. Furthermore, it is assumed that stakers will perform due diligence on the operators they delegate to.

## Recommendations

Consider informing users on this particular edge case in the documentation. Furthermore, provide plenty of warning to stakers on the front-end when operators may be considered malicious and have an extremely low `maxMagnitude`.

## Resolution

The EigenLayer team has added a Natspec comment to the `_getSlashingFactor()` function to inform users of this particular edge case. This issue has been resolved in PR #1089.

---
### Example 3

**Auto Label:** Inadequate slashing penalty calculation and distribution due to flawed balance scaling and improper share tracking, enabling stakers to evade penalties and misallocate penalties across cohorts.  

**Original Text Preview:**

## Description

The _increaseDelegation() function incorrectly assumes that the addedShares parameter corresponds to the number of added withdrawable shares. This leads to an incorrect deposit scaling factor (dsf) and operator shares update when the staker has been slashed on the beacon chain before delegating.

When a staker delegates to an operator, their depositedShares are used as the addedShares parameter in the _increaseDelegation() function:

```solidity
DelegationManager.sol::._delegate()
function _delegate(address staker, address operator) internal onlyWhenNotPaused(PAUSED_NEW_DELEGATION) {
    // ...
    // read staker's deposited shares and strategies to add to operator's shares
    // and also update the staker depositScalingFactor for each strategy
    (IStrategy[] memory strategies, uint256[] memory depositedShares) = getDepositedShares(staker);
    uint256[] memory slashingFactors = _getSlashingFactors(staker, operator, strategies);
    for (uint256 i = 0; i < strategies.length; ++i) {
        // forgefmt: disable-next-item
        _increaseDelegation({
            operator: operator,
            staker: staker,
            strategy: strategies[i],
            prevDepositShares: uint256(0),
            // @audit incorrectly assumes depositedShares = withdrawableShares
            addedShares: depositedShares[i],
            slashingFactor: slashingFactors[i]
        });
    }
}
```

The _increaseDelegation() function assumes that addedShares corresponds to withdrawable shares, which is the case for token strategies as no slashing has occurred. However, for `beaconChainETHStrategy`, it is possible for beacon chain slashing to occur before the staker is delegated, such that the assumption no longer holds true. This results in the operator being credited with more shares than they are entitled to.

```solidity
DelegationManager.sol::._increaseDelegation()
function _increaseDelegation(
    address operator,
    address staker,
    IStrategy strategy,
    uint256 prevDepositShares,
    uint256 addedShares, // @audit `addedShares` corresponds to an increase in withdrawable shares
    uint256 slashingFactor
) internal {
    // Ensure that the operator has not been fully slashed for a strategy
    // and that the staker has not been fully slashed if it is the beaconChainStrategy
    // This is to prevent a divWad by 0 when updating the depositScalingFactor
    require(slashingFactor != 0, FullySlashed());
    
    // Update the staker's depositScalingFactor. This only results in an update
    // if the slashing factor has changed for this strategy.
    DepositScalingFactor storage dsf = _depositScalingFactor[staker][strategy];
    // @audit dsf is incorrectly updated with `addedShares`
    dsf.update(prevDepositShares, addedShares, slashingFactor);
    
    emit DepositScalingFactorUpdated(staker, strategy, dsf.scalingFactor());
    
    // If the staker is delegated to an operator, update the operator's shares
    if (isDelegated(staker)) {
        // @audit operator shares is incorrectly increased by `addedShares`
        operatorShares[operator][strategy] += addedShares;
        emit OperatorSharesIncreased(operator, staker, strategy, addedShares);
    }
}
```

Furthermore, a staker that has been slashed on the beacon chain before delegation will be able to negate any slashing events prior to delegation. This is because the dsf is reset to 1 / slashingFactor on the initial deposit/delegation:

```solidity
SlashingLib.sol::update()
function update(
    DepositScalingFactor storage dsf,
    uint256 prevDepositShares,
    uint256 addedShares,
    uint256 slashingFactor
) internal {
    // If this is the staker's first deposit, set the scaling factor to
    // the inverse of slashingFactor
    if (prevDepositShares == 0) {
        // @audit this cancels out the slashingFactor, so any slashing events prior to delegation are negated
        dsf._scalingFactor = uint256(WAD).divWad(slashingFactor);
        return;
    }
    // ...
}
```

The actual issue has a high impact as it allows a staker to negate changes to their `beaconChainSlashingFactor` before delegating, and allows them to have more shares delegated to the operator than they are entitled to. This issue has an informational severity rating in the report as it was discovered by the EigenLayer team during the engagement.

## Recommendations

Consider implementing the following:
1. Scale `addedShares` by the `beaconChainSlashingFactor` for the `beaconChainETHStrategy` in the `_delegate()` function before calling `_increaseDelegation()`.
2. For the initial deposit/delegation, assign the dsf to 1 / maxMagnitude instead of 1 / slashingFactor.

## Resolution

The EigenLayer team has implemented the following:
1. Use `withdrawableShares` instead of `depositedShares` as the `addedShares` argument in `_increaseDelegation()`.
2. For the initial deposit/delegation, scale the current dsf by 1 / `operatorSlashingFactor` to selectively negate AVS slashing but maintain beacon chain slashing.

This issue has been resolved in PR #1045.

---
