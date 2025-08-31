# Cluster -1391

**Rank:** #359  
**Count:** 17  

## Label
Improper conditional logic and division-by-zero lead to incorrect gauge point calculations or transaction reverts, disrupting reward distribution and enabling denial-of-service in governance progression.

## Cluster Information
- **Total Findings:** 17

## Examples

### Example 1

**Auto Label:** Inconsistent rate enforcement and improper state updates lead to exploitable logic flaws, gas exhaustion, and inaccurate reward accounting under varying conditions.  

**Original Text Preview:**

## Severity

Low Risk

## Description

The `updateReward()` modifier is called in `deligateStake()`, `unstake()`, `fullyClaimReward()`, `_claimReward()`, `notifyRewardAmount()`.

At the moment this modifier loops through all rewards \* all locks, reads from storage, performs calculations, and writes to storage. These are gas-intensive operations that scale with the supported rewards and locks. If we have 10 rewards and 5 locks we get 50 loops in total just before the main function is about to get executed.

## Location of Affected Code

File: [contracts/StakingKo.sol](https://github.com/Beraji-Labs/staking-ko/blob/290103f27365cc419fe0a934feaf5f18e6e12d3a/contracts/StakingKo.sol)

## Impact

As there is no limit to how many reward tokens and locks will be added, this can result in DOS for the functions that use the `updateReward()` modifier due to gas reaching the block gas limit or becoming more expensive.

## Recommendation

Consider limiting the amount of locks and rewards in the contract.

## Team Response

Acknowledged.

---
### Example 2

**Auto Label:** Inconsistent rate enforcement and improper state updates lead to exploitable logic flaws, gas exhaustion, and inaccurate reward accounting under varying conditions.  

**Original Text Preview:**

## Severity

High Risk

## Description

The `fullyClaimReward()` function loops through all locks and all rewards for a pool. Let's say we have 10 reward tokens and 5 locks.

This is 10 reward tokens \* 5 locks = 50 loops.

In the `updateReward()` modifier we do the same 50 more loops.

It looks like `_claimReward()` incorrectly has the `updateReward` modifier which makes the looping exponential:

1. 50 loops in `updateReward()` before the execution of `fullyClaimReward()`.
2. 50 loops of `_claimReward()` _ 50 loops for each `_claimReward()` due to `updateReward()` modifier: 50 _ 50 = 2500 loops
3. 50 + 2500 = 2550 loops for calling `fullyClaimReward()` once.

This at some point could reach the block gas limit and prevent the user from calling `fullyClaimReward()` successfully.

## Location of Affected Code

File: [contracts/StakingKo.sol](https://github.com/Beraji-Labs/staking-ko/blob/290103f27365cc419fe0a934feaf5f18e6e12d3a/contracts/StakingKo.sol)

```solidity
function _claimReward(
    uint256 _poolId,
    uint256 _lockId,
    uint256 _rewardId
)
    internal
    validPool(_poolId)
    validPoolLock(_poolId, _lockId)
    validReward(_poolId, _rewardId)
@>  updateReward(_poolId, msg.sender)
{
    uint256 rwAmount = userRewards[_poolId][_lockId][_rewardId][msg.sender];
    PoolReward storage rw = mapPoolRewards[_poolId][_rewardId];

    if (rwAmount != 0) {
        userRewards[_poolId][_lockId][_rewardId][msg.sender] = 0;
        rw.rewardToken.safeTransfer(msg.sender, rwAmount);
        rw.balance -= rwAmount;
        emit RewardClaimed(msg.sender, _poolId, _lockId, _rewardId, rwAmount);
    }
}
```

## Impact

The `fullyClaimReward()` function can get DOSed or very expensive gas-wise.

## Recommendation

Remove `updateReward()` modifier from `_claimReward()`.

## Team Response

Fixed.

---
### Example 3

**Auto Label:** Inconsistent rate enforcement and improper state updates lead to exploitable logic flaws, gas exhaustion, and inaccurate reward accounting under varying conditions.  

**Original Text Preview:**

**Description:** `DropBox::claimDropBoxes` allows users to pass an array of `boxIds` to claim per transaction while charging a constant `claimFee` per transaction:
```solidity
// Require that the user sends a fee of "claimFee" amount
if (msg.value != claimFee) revert InvalidClaimFee();
```

This means that (not including gas fees):
* claiming 1 box costs the same as claiming 100 boxes, if done in the 1 transaction
* claiming 100 boxes 1-by-1 costs 100x more than claiming 100 boxes in 1 transaction

**Impact:** The protocol will collect:
* less fees if users claim batches of boxes
* more fees if users claim boxes one-by-one

**Recommended Mitigation:** Charge `claimFee` per transaction taking into account the number of boxes being claimed, eg:
```solidity
// Require that the user sends a fee of "claimFee" amount per box
if (msg.value != claimFee * _boxIds.length) revert InvalidClaimFee();
```

**Mode:**
Acknowledged; this was required by the product specification.

---
