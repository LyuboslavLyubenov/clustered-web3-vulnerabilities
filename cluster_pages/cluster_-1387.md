# Cluster -1387

**Rank:** #259  
**Count:** 34  

## Label
Failure to validate time-based eligibility or cycle progression enables users to claim rewards prematurely or stake after periods end, trapping rewards and undermining fair distribution and staking integrity.

## Cluster Information
- **Total Findings:** 34

## Examples

### Example 1

**Auto Label:** Inconsistent reward timing and epoch handling lead to improper reward distribution, missed claims, and potential reward over-delivery or denial-of-service.  

**Original Text Preview:**

We have two different kinds of gauges according to different pools. When we notify rewards, the reward rate's calculation way is different.

In gauge, we will use `(_amount * PRECISION) / DURATION`. In CLGauge, we will use `rewardRate = amount / epochDurationLeft;`. The `epochDurationLeft` may be less than 1 week, depending on the different timestamp.

In voter, we have one function `distro`. It's possible that all gauges' rewards will be distributed in one transaction. This will cause that in CL Gauges, in the next 7 days, we will distribute rewards in one time slot, and in another time slot, we will not distribute any reward.

```solidity
    function notifyRewardAmount(
        uint256 _amount
    ) external onlyVoterOrAuthorized updateReward(address(0)) {
        _claimFees();
        // reword token is kitten.
        kitten.safeTransferFrom(msg.sender, address(this), _amount);
        // If last period is finished, we will start one new period.
        if (block.timestamp >= finishAt) {
            // PRECISION = 1e18
            rewardRate = (_amount * PRECISION) / DURATION;
        }
    }
    function notifyRewardAmount(uint256 amount) external nonReentrant {
        require(amount > 0);
        require(msg.sender == address(voter));
        _claimFees();
        // When we notify rewards, we need to update or accrue the previous rewards.
        pool.updateRewardsGrowthGlobal();
        address msgSender = msg.sender;
        uint256 timestamp = block.timestamp;
        // epoch remaining time.
        // epochDurationLeft may be less than 1 week.
        uint256 epochDurationLeft = ProtocolTimeLibrary.epochNext(timestamp) -
            timestamp;
        if (block.timestamp >= periodFinish) {
            rewardRate = amount / epochDurationLeft;
            pool.syncReward({
                rewardRate: rewardRate,
                rewardReserve: amount,
                periodFinish: nextPeriodFinish
            });
        }
    }
```

Recommendation: Suggest using a similar formula for the rewardRate calculation.

---
### Example 2

**Auto Label:** Premature reward claims due to insufficient period validation and flawed state timing, enabling attackers to extract rewards before distribution and disrupt reward integrity.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** High

## Description

In VotingReward, voters can gain some rewards because of their vote. There are two kind of possible rewards in VotingReward.

1. Kitten token from voter.
2. Users can add some incentive tokens via function `incentivize`.

If users incentive some rewards via function `incentivize`, we will record these rewards in the next period. Users can vote for the related pool to get some rewards.

The problem here is that we miss one `_period` validation in function `getRewardForPeriod`. Users can get the future period's rewards.

Let's consider below scenario:

1. Alice incentives 500 USDC for next period, next period = X.
2. Bob as the first voter, votes for the related pool.
3. Bob gets rewards for the period X immediately. Currently, Bob is the only person who vote for this pool in this period. Bob will get all the rewards.

```solidity
    function incentivize(
        address _token,
        uint256 _amount
    ) external virtual nonReentrant {
        // Here we have one white list for token. So we cannot manipulate the token.
        if (voter.isWhitelisted(_token) == false)
            revert NotWhitelistedRewardToken();
        uint256 currentPeriod = getCurrentPeriod() + 1;
        uint256 amount = _addReward(currentPeriod, _token, _amount);
    }
    function _deposit(uint256 _amount, uint256 _tokenId) external onlyVoter {
        uint256 nextPeriod = getCurrentPeriod() + 1;

        tokenIdVotesInPeriod[nextPeriod][_tokenId] += _amount;
        totalVotesInPeriod[nextPeriod] += _amount;
    }
    function getRewardForPeriod(
        uint256 _period,
        uint256 _tokenId,
        address _token
    ) external nonReentrant {
        // Only owner or approved can get rewards.
        if (!veKitten.isApprovedOrOwner(msg.sender, _tokenId))
            revert NotApprovedOrOwner();
        _getReward(_period, _tokenId, _token, msg.sender);
    }
```

## Recommendations

Add one input parameter check, don't allow to get a reward from one future period.

---
### Example 3

**Auto Label:** Timestamp-based epoch boundary flaws lead to reward double-claiming, improper state tracking, and incorrect event signaling, causing reward depletion or loss for users.  

**Original Text Preview:**

In the `ExternalBribe.earned` function, users can permanently lose rewards for an epoch if they call `getReward` exactly at the epoch's end time.

Consider a scenario when a user has one checkpoint and calls `getReward` when `block.timestamp` is exactly equal to `_lastEpochEnd`:

- The condition `block.timestamp > _lastEpochEnd` evaluates to `false`.
- The reward calculation is skipped, returning 0.
- The `getReward` function updates `lastEarn[token][tokenId]` to `block.timestamp` (which is `_lastEpochEnd`).
- When the user tries again later (for example `block.timestamp + 1`), `_startTimestamp` will be set to `lastEarn[token][tokenId]` (which equals to `_lastEpochEnd`).
- The condition `_startTimestamp < _lastEpochEnd` will now evaluate to `false`.
- The user permanently loses the rewards for that epoch.

This creates a timing vulnerability where users lose their rewards.

```solidity
    function earned(address token, uint tokenId) public view returns (uint) {
        uint _startTimestamp = lastEarn[token][tokenId];
        ...
        Checkpoint memory _cp0 = checkpoints[tokenId][_endIndex];
        (_prev._prevTs, _prev._prevBal) = (_cp0.timestamp, _cp0.balanceOf);

        uint _lastEpochStart = _bribeStart(_prev._prevTs);
        uint _lastEpochEnd = _lastEpochStart + DURATION;

        if (
            block.timestamp > _lastEpochEnd && _startTimestamp < _lastEpochEnd // @audit check will not pass if user calls getReward at _lastEpochEnd
        ) {
            SupplyCheckpoint memory _scp0 = supplyCheckpoints[
                getPriorSupplyIndex(_lastEpochEnd)
            ];
            _prev._prevSupply = _scp0.supply;

            reward +=
                (_prev._prevBal *
                    tokenRewardsPerEpoch[token][_lastEpochStart]) /
                _prev._prevSupply;
        }

        return reward;
    }
```

**Recommendations**

Revert when users call `getReward` at epoch end to avoid losing rewards.

---
