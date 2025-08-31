# Cluster -1081

**Rank:** #75  
**Count:** 222  

## Label
Inadequate arithmetic precision and improper state management enable attackers to manipulate token allocations, bypass vesting, or exploit remainders for unfair gains, leading to financial loss or unfair advantages.

## Cluster Information
- **Total Findings:** 222

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

**Auto Label:** Flawed state synchronization and improper validation lead to incorrect asset balances, misrouted fees, and unauthorized access, enabling attacks, DoS, and economic exploitation.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** High

## Description

Due to the lack of a check on the split amount inside `split`, a user can provide an arbitrary `_amount` to the function. Since `locked.amount` is stored as an `int128`, this can result in one of the split tokens having a negative value, while the other is set to the full `_amount` provided.

```solidity
    function split(
        uint _from,
        uint _amount
    ) external returns (uint _tokenId1, uint _tokenId2) {
        address msgSender = msg.sender;

        require(_isApprovedOrOwner(msgSender, _from));
        require(attachments[_from] == 0 && !voted[_from], "attached");
        require(_amount > 0, "Zero Split");

        // burn old NFT
        LockedBalance memory _locked = locked[_from];
        int128 value = _locked.amount;
        locked[_from] = LockedBalance(0, 0);
        _checkpoint(_from, _locked, LockedBalance(0, 0));
        _burn(_from);

        // set max lock on new NFTs
        _locked.end = block.timestamp + MAXTIME;

        int128 _splitAmount = int128(uint128(_amount));
        // @audit - underflow is possible
>>>     _locked.amount = value - _splitAmount; // already checks for underflow here in ^0.8.0
        _tokenId1 = _createSplitNFT(msgSender, _locked);

>>>     _locked.amount = _splitAmount;
        _tokenId2 = _createSplitNFT(msgSender, _locked);

        emit Split(
            _from,
            _tokenId1,
            _tokenId2,
            msgSender,
            uint(uint128(locked[_tokenId1].amount)),
            uint(uint128(_splitAmount)),
            _locked.end,
            block.timestamp
        );
    }
```

This can be abused by initially depositing a small amount of tokens, then calling `split` with an arbitrary amount to create a large `locked` data. Then can be withdrawn at the end of the lock period, and also grants an arbitrary amount of voting power and balance, which can be used to claim rewards from gauges and bribes.

PoC :

```solidity
    function testSplitVeKittenUnderflow() public {
        testDistributeVeKitten();

        vm.startPrank(user1);

        uint veKittenId = veKitten.tokenOfOwnerByIndex(user1, 0);
        (int128 lockedAmount, uint endTime) = veKitten.locked(veKittenId);

        (uint t1, uint t2) = veKitten.split(veKittenId, uint256(uint128(lockedAmount)) * 100);

        (int128 lockedAmountSplit, ) = veKitten.locked(t2);

        console.log("initial token to split locked amount :");
        console.log(lockedAmount);
        console.log("splitted token locked amount :");
        console.log(lockedAmountSplit);

        vm.stopPrank();
    }
```

Log output :

```shell
Logs:
  initial token to split locked amount :
  100000000000000000000000000
  splitted token locked amount :
  10000000000000000000000000000
```

## Recommendations

Validate that the provided `_amount` does not exceed the locked amount of the split token.

---
