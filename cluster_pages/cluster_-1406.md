# Cluster -1406

**Rank:** #414  
**Count:** 11  

## Label
Timestamp-based epoch boundary flaws lead to reward double-claiming, improper state tracking, and incorrect event signaling, causing reward depletion or loss for users.

## Cluster Information
- **Total Findings:** 11

## Examples

### Example 1

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
### Example 2

**Auto Label:** Timestamp-based epoch boundary flaws lead to reward double-claiming, improper state tracking, and incorrect event signaling, causing reward depletion or loss for users.  

**Original Text Preview:**

The getEpochStart function in the ExternalBribe contract contains redundant logic that can lead to unnecessary complexity.

In the `ExternalBribe` contract, the `getEpochStart` function is defined as follows:

```solidity
function getEpochStart(uint timestamp) public pure returns (uint) {
    uint bribeStart = _bribeStart(timestamp);
    uint bribeEnd = bribeStart + DURATION;
    return timestamp < bribeEnd ? bribeStart : bribeStart + 7 days;
}
```

The issue arises because the calculation of `bribeStart + 7` days will always be greater than `timestamp` if timestamp is within the current epoch. This means that the condition checking and the alternative return value are redundant.

---
### Example 3

**Auto Label:** Timestamp-based epoch boundary flaws lead to reward double-claiming, improper state tracking, and incorrect event signaling, causing reward depletion or loss for users.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

In the `ExternalBribe` contract, a malicious user can claim rewards twice for the same `tokenId` in the same epoch due to flawed logic in the `ExternalBribe::earned()` function. This issue allows a user to receive rewards multiple times from a single epoch, which can result in other users being unable to claim their rightful rewards due to reward depletion.

The problem lies in the update of the `prevRewards.timestamp` field inside the loop. The contract updates `prevRewards.timestamp` **unconditionally**, even when `_nextEpochStart <= prevRewards.timestamp`. This causes reward accumulation logic to break, allowing rewards from a previously claimed epoch to be included again.

`ExternalBribe::earned()` function:

```solidity
function earned(address token, uint tokenId) public view returns (uint) {
    ...
    uint _startIndex = getPriorBalanceIndex(tokenId, _startTimestamp);
    uint _endIndex = numCheckpoints[tokenId] - 1;

    uint reward = 0;
    // you only earn once per epoch (after it's over)
    RewardCheckpoint memory prevRewards;
    prevRewards.timestamp = _bribeStart(_startTimestamp);

    PrevData memory _prev;
    _prev._prevSupply = 1;

    if (_endIndex > 0) {
        for (uint i = _startIndex; i <= _endIndex - 1; i++) {
            _prev._prevTs = checkpoints[tokenId][i].timestamp;
            _prev._prevBal = checkpoints[tokenId][i].balanceOf;

            uint _nextEpochStart = _bribeStart(_prev._prevTs);
            if (_nextEpochStart > prevRewards.timestamp) {
                reward += prevRewards.balance;
            }

=>          prevRewards.timestamp = _nextEpochStart;
            _prev._prevSupply = supplyCheckpoints[getPriorSupplyIndex(_nextEpochStart + DURATION)].supply;

            prevRewards.balance =
                (_prev._prevBal * tokenRewardsPerEpoch[token][_nextEpochStart]) /
                _prev._prevSupply;
        }
    }
    ...
    return reward;
}
```

An example scenario:

We'll refer to the `tokenId` as "User A".

1. **Epoch 1**:

   - User A deposits → Checkpoint 1 is created.

2. **Epoch 2**:

   - User A claims rewards → Earns Epoch 1 reward → `lastEarn` updated (green point).

3. **Epoch 2**:

   - User A makes two more deposits → Checkpoints 2 and 3 are created.

4. User A claims again:

   - `_startIndex`: Checkpoint 1.
   - `_endIndex`: Checkpoint 3.
   - `prevRewards.timestamp`: Start of Epoch 2.

- **Iteration 1** (Checkpoint 1):

  - `_nextEpochStart = Start of Epoch 1`.
  - `prevRewards.timestamp = Start of Epoch 1`.
  - `prevRewards.balance` updated (Epoch 1 reward).

- **Iteration 2** (Checkpoint 2):

  - `_nextEpochStart = Start of Epoch 2`.
  - Since `_nextEpochStart > prevRewards.timestamp`, `reward += Epoch 1 reward`.
  - `prevRewards.timestamp = Start of Epoch 2`.

- **Checkpoint 3** is skipped or only used depending on current timestamp.

5. Result: User A successfully reclaimed already claimed rewards.

## Recommendations

Update the logic inside `ExternalBribe::earned()` so that `prevRewards.timestamp` is only updated when `_nextEpochStart > prevRewards.timestamp`.

---
