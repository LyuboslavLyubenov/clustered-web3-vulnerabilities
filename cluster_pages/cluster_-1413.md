# Cluster -1413

**Rank:** #312  
**Count:** 23  

## Label
Inaccurate reward calculations due to unvalidated state updates or improper balance tracking, enabling attackers to manipulate reward pools or induce unfair token distributions.

## Cluster Information
- **Total Findings:** 23

## Examples

### Example 1

**Auto Label:** Out-of-gas and overflow vulnerabilities due to inefficient loops and unchecked reward accumulation, leading to transaction failures or incorrect reward distributions.  

**Original Text Preview:**

The Gauge contract generates a checkpoint for each user operation, which can lead to excessive gas consumption during reward distribution. If a user has not claimed rewards for an extended period, traversing through numerous checkpoints may result in an Out of Gas (OOG) error.

When a user claims rewards, the contract starts from the last checkpoint and iterates through all checkpoints associated with that user. If a user has not claimed rewards for a long time, the number of checkpoints can grow significantly, leading to increased gas costs during the iteration.

```solidity
    function earned(address token, address account) public view returns (uint) {
        uint _startTimestamp = Math.max(
            lastEarn[token][account],
            rewardPerTokenCheckpoints[token][0].timestamp
        );
        if (numCheckpoints[account] == 0) {
            return 0;
        }

        uint _startIndex = getPriorBalanceIndex(account, _startTimestamp);
        uint _endIndex = numCheckpoints[account] - 1;

        uint reward = 0;

        if (_endIndex > 0) {
            for (uint i = _startIndex; i <= _endIndex - 1; i++) {
                Checkpoint memory cp0 = checkpoints[account][i];
                Checkpoint memory cp1 = checkpoints[account][i + 1];
                (uint _rewardPerTokenStored0, ) = getPriorRewardPerToken(
                    token,
                    cp0.timestamp
                );
                (uint _rewardPerTokenStored1, ) = getPriorRewardPerToken(
                    token,
                    cp1.timestamp
                );
                reward +=
                    (cp0.balanceOf *
                        (_rewardPerTokenStored1 - _rewardPerTokenStored0)) /
                    PRECISION;
            }
        }

        Checkpoint memory cp = checkpoints[account][_endIndex];
        (uint _rewardPerTokenStored, ) = getPriorRewardPerToken(
            token,
            cp.timestamp
        );
        reward +=
            (cp.balanceOf *
                (rewardPerToken(token) -
                    Math.max(
                        _rewardPerTokenStored,
                        userRewardPerTokenStored[token][account]
                    ))) /
            PRECISION;

        return reward;
    }
```

This could potentially cause the transaction to run out of gas, resulting in a failed operation.

Recommendations:
Allow users to claim rewards for special checkpoints.

---
### Example 2

**Auto Label:** Out-of-gas and overflow vulnerabilities due to inefficient loops and unchecked reward accumulation, leading to transaction failures or incorrect reward distributions.  

**Original Text Preview:**

The farm-manager contract has a static emergency unlock penalty (initialized to 2% in the deployment file `deploy_mantra_dex.sh`) regardless of the position’s unlocking duration. However, longer unlocking durations provide significantly higher reward weight multipliers (up to 16x for 1 year lockups). This creates an economic imbalance where users are incentivized to:

1. Create positions with maximum unlocking duration to get the highest weight multiplier (up to 16x).
2. Emergency unlock when they want liquidity, only paying the fixed 2% penalty.

This undermines the intended lockup mechanism since users can get much higher rewards while maintaining effective liquidity through emergency unlocks. The impact is that the protocol’s liquidity stability guarantees are weakened, as users are economically incentivized to game the system rather than maintain their intended lock periods.

Moreover, it can be profitable to open a huge position 1 second before an epoch ends and withdraw immediately (in the new epoch), which hurts real users of the system.

### Proof of Concept

The key components of this issue are:

1. Emergency unlock penalty is static, set during initialization [here](https://github.com/code-423n4/2024-11-mantra-dex/blob/26714ea59dab7ecfafca9db1138d60adcf513588/contracts/farm-manager/src/position/commands.rs# L322).
2. Weight calculation increases significantly with lock duration [here](https://github.com/code-423n4/2024-11-mantra-dex/blob/26714ea59dab7ecfafca9db1138d60adcf513588/contracts/farm-manager/src/position/helpers.rs# L28-L65).

For example:
```

#[test]
fn test_calculate_weight() {
    // 1 day lockup = 1x multiplier
    let weight = calculate_weight(&coin(100, "uwhale"), 86400u64).unwrap();
    assert_eq!(weight, Uint128::new(100));

    // 1 year lockup = ~16x multiplier
    let weight = calculate_weight(&coin(100, "uwhale"), 31556926).unwrap();
    assert_eq!(weight, Uint128::new(1599));
}
```

Consider this scenario:

1. User creates position with 1000 tokens and 1 year lock, getting `~16x` weight (16000).
2. After collecting boosted rewards, user emergency exits paying only 2% (20 tokens).
3. Net result: User got 16x rewards while maintaining effective liquidity with minimal penalty.

This makes it economically optimal to always choose maximum duration and emergency unlock when needed, defeating the purpose of tiered lockup periods.

### Recommended mitigation steps

The emergency unlock penalty should scale with:

1. Remaining lock duration.
2. Position’s weight multiplier.

Suggested formula:
```

emergency_penalty = base_penalty * (remaining_duration / total_duration) * (position_weight / base_weight)
```

This would make emergency unlocks proportionally expensive for positions with higher weights and longer remaining durations, better aligning incentives with the protocol’s goals.

Alternative mitigations:

* Cap maximum unlock duration to reduce exploitability.
* Increase base emergency unlock penalty.
* Add minimum hold period before emergency unlocks are allowed.

The key is ensuring the penalty properly counterbalances the increased rewards from longer lock periods.

**[jvr0x (MANTRA) confirmed and commented](https://code4rena.com/audits/2024-11-mantra-dex/submissions/F-110?commentParent=gSq3hpwTeig):**

> This is a valid concern and the team is aware of that. However, the way the team intends to instantiate the farm manager at the beginning is to set min and max unlock periods to 1 day, so in that case all positions would be the same.
>
> Will probably address this issue once/if the team decides to increase the constraint.

---

---
### Example 3

**Auto Label:** Out-of-gas and overflow vulnerabilities due to inefficient loops and unchecked reward accumulation, leading to transaction failures or incorrect reward distributions.  

**Original Text Preview:**

[/contracts/farm-manager/src/farm/commands.rs# L43-L94](https://github.com/code-423n4/2024-11-mantra-dex/blob/26714ea59dab7ecfafca9db1138d60adcf513588/contracts/farm-manager/src/farm/commands.rs# L43-L94)

### Finding description and impact

The `claim` function iterates over the user positions and calculates the rewards in nested loops. The issue is that every blockchain, to combat against gas attacks of infinite loops, has a block gas limit. If this limit is exceeded, that transaction cannot be included in the chain. The implementation of the `claim` function here is of the order of `N^3` and is thus highly susceptible to an out of gas error.

The claim function iterates over all the user’s positions.
```

let lp_denoms = get_unique_lp_asset_denoms_from_positions(open_positions);

for lp_denom in &lp_denoms {
    // calculate the rewards for the lp denom
    let rewards_response = calculate_rewards(
        deps.as_ref(),
        &env,
        lp_denom,
        &info.sender,
        current_epoch.id,
        true,
    )?;
    //...
}
```

Lets say the user has `P` positions, all of different `lp_deonm` values. Thus this loop is of the order of `P`. The `calculate_rewards` function then loops over all the farms of each `lp_denom`.
```

let farms = get_farms_by_lp_denom(
    deps.storage,
    lp_denom,
    None,
    Some(config.max_concurrent_farms),
)?;
//...
for farm in farms {
    // skip farms that have not started
    if farm.start_epoch > current_epoch_id {
        continue;
    }

    // compute where the user can start claiming rewards for the farm
    let start_from_epoch = compute_start_from_epoch_for_address(
        deps.storage,
        &farm.lp_denom,
        last_claimed_epoch_for_user,
        receiver,
    )?;
    //...
}
```

Say there are `F` farms, then this inner loop is of the order of `F`. Then for each farm, the reward is calculated by iterating over all the epochs from `start_from_epoch` up to the `current_epoch`.
```

for epoch_id in start_from_epoch..=until_epoch {
    if farm.start_epoch > epoch_id {
        continue;
    }
    //...
}
```

The `start_from_epoch` can be the very first deposit of the user, far back in time, if this is the first time the user is claiming rewards. Thus, this loop can run very long if the position is years old. Say the epoch loop is of the order of `E`.

Since these 3 loops are nested, the `claim` function is of the order of `P*F*E`. `P` and `F` are restricted by the config can can have maximum values of the order of 10. But `E` can be very large, and is actually the order of epoch number. So if epochs are only a few days long, the `E` can be of the order of 500 over a couple of years.

Thus the `claim` function can be of the order of `50_000`. This is an issue since it requires a loop running `50_000` times along with reward calculations and even token transfers. This can be above the block gas limit and thus the transaction will fail.

There is no functionality to skip positions/farms/epochs. Thus users cannot claim rewards of only a few particular farms or epochs. This part of the code is also executed during the `close_position` function, which checks if rewards are 0. Thus, the `close_position` function can also fail due to the same issue, and users are thus forced to emergency withdraw and lose deposits as well as their rewards.

Thus users who join a bunch of different farms and keep their positions for a long time can hit the block gsa limit during the time of claiming rewards or closing positions.

The OOG issue due to large nesting depth is present in multiple instances in the code, this is only one example.

### Proof of Concept

`P`, the number of open positions of a user, is restricted by limit of 100 stored in `MAX_ITEMS_LIMIT`. `F` is restricted by the max concurrent no of farms per `lp_denom`, which we can assume to be 10. `E` is of the order of epochs between the first deposit and the current epoch, which can be in the 100s if epochs are single days, or 100s if epochs are weeks.

Thus, `P*F*E` is of the order of `100*10*100 = 100_000`; `100_000` iterations are required for the `claim` function on top of token transfers and math calculations. This can easily exceed the block gas limit.

### Recommended mitigation steps

The order of the nested loops need to be decreased. This can be done in multiple ways.

1. Implement sushi-masterchef style reward accounting. This way the entire `E` number of epochs dont need to be looped over.
2. Implement a way to only process a given number of positions. This way `P` can also be restricted and users can claim in batches.

**jvr0x (MANTRA) confirmed**

---

---
