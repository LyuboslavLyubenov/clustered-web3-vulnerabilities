# Cluster -1214

**Rank:** #246  
**Count:** 38  

## Label
Inconsistent eligibility and disqualification logic across contracts, leading to incorrect state validation, transaction reverts, and flawed reward distribution due to improper state checks and missing race condition handling.

## Cluster Information
- **Total Findings:** 38

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

**Auto Label:** Inaccurate state validation leading to incorrect eligibility, stake attribution, and reward distribution due to flawed boundary conditions and missing identity checks.  

**Original Text Preview:**

##### Description

In the `getDelegator` function of the **PledgeAgent** contract, there is a potential for inconsistent data updates. The current implementation only updates values when `realtimeAmount`is non-zero, potentially missing scenarios where `stakedAmount`is non-zero but `realtimeAmount`is zero. Additionally, directly assigning `cd.changeRound` to `changeRound`might overwrite a more recent value, leading to incorrect round tracking. While these scenarios have a low probability of occurring due to typical protocol behavior, they are worth mentioning to ensure robustness in edge cases.

  

Code Location
-------------

```
  function getDelegator(address agent, address delegator) external view returns (CoinDelegator memory cd) {
      cd = agentsMap[agent].cDelegatorMap[delegator];
      (bool success, bytes memory result) = CORE_AGENT_ADDR.staticcall(abi.encodeWithSignature("getDelegator(address,address)", agent, delegator));
      require (success, "call CORE_AGENT_ADDR.getDelegator() failed");
      (uint256 stakedAmount, uint256 realtimeAmount, uint256 transferredAmount, uint256 changeRound) = abi.decode(result, (uint256,uint256,uint256,uint256));
      if (realtimeAmount != 0) {
        cd.deposit = cd.newDeposit + stakedAmount;
        cd.newDeposit += realtimeAmount;
        cd.changeRound = changeRound;
        cd.transferOutDeposit = transferredAmount;
        cd.transferInDeposit = 0;
      }
  }
```

##### BVSS

[AO:A/AC:L/AX:H/R:N/S:U/C:N/A:N/I:L/D:N/Y:N (0.8)](/bvss?q=AO:A/AC:L/AX:H/R:N/S:U/C:N/A:N/I:L/D:N/Y:N)

##### Recommendation

Modify the logic to update values when either `stakedAmount`or `realtimeAmount`is non-zero. Use `cd.changeRound = max(cd.changeRound, changeRound)` to ensure the most recent round is retained, enhancing data consistency and system reliability.

##### Remediation

**SOLVED**: The **CoreDAO team** solved the issue in the specified commit id. They also stated the following:

> *It is not necessary to* `max(cd.changeRound, changeRound)` *because* `changeRound` *is fetched from COREAgent.sol which is definitely >= the value left in PledgeAgent which has not been changed since data moved to COREAgent.sol.*

##### Remediation Hash

<https://github.com/coredao-org/core-genesis-contract/commit/6f7efc313ee1491a5c0ca58fe1160becca67baa8>

---
