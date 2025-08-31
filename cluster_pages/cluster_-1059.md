# Cluster -1059

**Rank:** #135  
**Count:** 104  

## Label
Underflow vulnerabilities in unsigned integer arithmetic, triggered by improper bounds checking and price/asset fluctuations, leading to reverts, stalled withdrawals, or incorrect state calculations with significant operational and financial impacts.

## Cluster Information
- **Total Findings:** 104

## Examples

### Example 1

**Auto Label:** Arithmetic underflows and overflows in critical calculations lead to incorrect state updates, reward miscomputations, or reverts, compromising user claims and system integrity.  

**Original Text Preview:**

Inside the `earned` function, it queries the latest `getRewardGrowthInside` and then calculates the `rewardGrowthInsideDelta`.

```solidity
    function earned(uint256 nfpTokenId) public view returns (uint) {
        uint256 timeDelta = block.timestamp - pool.lastUpdated();

        uint256 rewardGrowthGlobalX128 = pool.rewardGrowthGlobalX128();
        uint256 rewardReserve = pool.rewardReserve();
        if (timeDelta != 0 && rewardReserve > 0 && pool.stakedLiquidity() > 0) {
            uint256 reward = rewardRate * timeDelta;
            if (reward > rewardReserve) reward = rewardReserve;

            rewardGrowthGlobalX128 +=
                (reward * FixedPoint128.Q128) /
                pool.stakedLiquidity();
        }
        (
            ,
            ,
            ,
            ,
            ,
            int24 _tickLower,
            int24 _tickUpper,
            uint128 _liquidity,
            ,
            ,
            ,

        ) = nfp.positions(nfpTokenId);

        uint256 rewardGrowthInsideInitial = rewardGrowthInside[nfpTokenId];
        uint256 rewardGrowthInsideCurrent = pool.getRewardGrowthInside(
            _tickLower,
            _tickUpper,
            rewardGrowthGlobalX128
        );

>>>     uint256 rewardGrowthInsideDelta = rewardGrowthInsideCurrent -
            rewardGrowthInsideInitial;
        return (rewardGrowthInsideDelta * _liquidity) / FixedPoint128.Q128;
    }
```

The calculation of the reward growth and delta implicitly relies on underflow, which will not work correctly because the current `CLGauge` uses Solidity version 0.8.

Detailed information about the root cause: https://github.com/Uniswap/v3-core/issues/573.

Recommendations:
Wrap the `rewardGrowthInsideDelta` inside `unchecked` block to allow underflow.

---
### Example 2

**Auto Label:** Arithmetic underflows and overflows in critical calculations lead to incorrect state updates, reward miscomputations, or reverts, compromising user claims and system integrity.  

**Original Text Preview:**

The `earned` function in CLGauge.sol performs several arithmetic calculations that could potentially overflow, causing the function to revert and leading to a Denial of Service condition for reward claims:

When computing updated reward growth: `rewardGrowthGlobalX128 += (reward * FixedPoint128.Q128) / pool.stakedLiquidity()`
And when calculating the final earned reward amount `(rewardGrowthInsideDelta * _liquidity) / FixedPoint128.Q128;`

In both cases, the intermediate multiplication could exceed the uint256 maximum value, causing a revert.
It leads to users will be unable to claim their rewards.

```solidity
    function earned(uint256 nfpTokenId) public view returns (uint) {
        ...
        if (timeDelta != 0 && rewardReserve > 0 && pool.stakedLiquidity() > 0) {
            uint256 reward = rewardRate * timeDelta;
            if (reward > rewardReserve) reward = rewardReserve;

            rewardGrowthGlobalX128 +=
                (reward * FixedPoint128.Q128) / // @audit revert due to overflow here
                pool.stakedLiquidity();
        }
        ...
        return (rewardGrowthInsideDelta * _liquidity) / FixedPoint128.Q128; // @audit revert due to overflow here
    }
```

It's recommended to use OpenZeppelin Math.mulDiv that can handles intermediate overflows.

Reference:
https://github.com/velodrome-finance/slipstream/blob/main/contracts/gauge/CLGauge.sol#L123.
https://github.com/velodrome-finance/slipstream/blob/main/contracts/gauge/CLGauge.sol#L132.

---
### Example 3

**Auto Label:** Arithmetic underflows and overflows in critical calculations lead to incorrect state updates, reward miscomputations, or reverts, compromising user claims and system integrity.  

**Original Text Preview:**

Inside the `earned` function, it queries the latest `getRewardGrowthInside` and then calculates the `rewardGrowthInsideDelta`.

```solidity
    function earned(uint256 nfpTokenId) public view returns (uint) {
        uint256 timeDelta = block.timestamp - pool.lastUpdated();

        uint256 rewardGrowthGlobalX128 = pool.rewardGrowthGlobalX128();
        uint256 rewardReserve = pool.rewardReserve();
        if (timeDelta != 0 && rewardReserve > 0 && pool.stakedLiquidity() > 0) {
            uint256 reward = rewardRate * timeDelta;
            if (reward > rewardReserve) reward = rewardReserve;

            rewardGrowthGlobalX128 +=
                (reward * FixedPoint128.Q128) /
                pool.stakedLiquidity();
        }
        (
            ,
            ,
            ,
            ,
            ,
            int24 _tickLower,
            int24 _tickUpper,
            uint128 _liquidity,
            ,
            ,
            ,

        ) = nfp.positions(nfpTokenId);

        uint256 rewardGrowthInsideInitial = rewardGrowthInside[nfpTokenId];
        uint256 rewardGrowthInsideCurrent = pool.getRewardGrowthInside(
            _tickLower,
            _tickUpper,
            rewardGrowthGlobalX128
        );

>>>     uint256 rewardGrowthInsideDelta = rewardGrowthInsideCurrent -
            rewardGrowthInsideInitial;
        return (rewardGrowthInsideDelta * _liquidity) / FixedPoint128.Q128;
    }
```

The calculation of the reward growth and delta implicitly relies on underflow, which will not work correctly because the current `CLGauge` uses Solidity version 0.8.

Detailed information about the root cause: https://github.com/Uniswap/v3-core/issues/573.

Recommendations:
Wrap the `rewardGrowthInsideDelta` inside `unchecked` block to allow underflow.

---
