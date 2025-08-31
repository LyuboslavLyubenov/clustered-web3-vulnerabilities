# Cluster -1426

**Rank:** #435  
**Count:** 9  

## Label
Inconsistent state synchronization across pool updates leads to flawed share tracking, incorrect reward calculations, and phantom or lost asset distributions, causing financial discrepancies and unfair user outcomes.

## Cluster Information
- **Total Findings:** 9

## Examples

### Example 1

**Auto Label:** Inconsistent ownership and share tracking due to failed state updates and missing transfer mechanisms, leading to unreliable rewards, impaired exits, and inaccurate share allocations under dynamic pricing.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Medium

## Description

When a Kodiak island contract is set in the `Burve`, on `mint()` a portion of the liquidity will be used to mint island shares, which will then be deposited into the station proxy at the name of the recipient, and the `islandSharesPerOwner` mapping will be updated for the recipient. The recipient will also receive `Burve` tokens (shares) for the liquidity provided to the Uniswap pools.

However, if the recipient transfers the `Burve` tokens to another address, the island shares are still assigned to him, both in `StationProxy` and in the `Burve`'s `islandSharesPerOwner` mapping. This means that after the transfer, the new owner of the `Burve` tokens will not be able to harvest the rewards in `StationProxy`, nor will he be able to burn the island shares in exchange for the underlying liquidity.

## Recommendations

Overwrite the `_update()` function so that on `Burve` transfers:

- The proportional amount of LP tokens are withdrawn from `StationProxy`.
- The LP tokens are deposited again in the name of the new owner.
- `islandSharesPerOwner` is updated by both the old and new owner.

---
### Example 2

**Auto Label:** Inconsistent ownership and share tracking due to failed state updates and missing transfer mechanisms, leading to unreliable rewards, impaired exits, and inaccurate share allocations under dynamic pricing.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

The Burve contract allows users to provide liquidity to both the Island pool and Uniswap V3 pools. The number of shares minted to users is determined using the following calculation:

```solidity
    function islandLiqToShares(uint128 liq) internal view returns (uint256 shares) {
        if (address(island) == address(0x0)) {
            revert NoIsland();
        }

        (uint160 sqrtRatioX96,,,,,,) = pool.slot0();

        (uint256 amount0, uint256 amount1) = getAmountsFromLiquidity(
            sqrtRatioX96,
            island.lowerTick(),
            island.upperTick(),
            liq
        );

        (,, shares) = island.getMintAmounts(amount0, amount1);
    }
```

Since the amount of shares depends on `sqrtRatioX96`, any fluctuations in this value will impact the number of shares users receive. As a result, users may end up with fewer tokens than expected.

## Recommendations

Consider adding a `minSharesOut` parameter to the mint/burn functions.

---
### Example 3

**Auto Label:** Inconsistent state synchronization across pool updates leads to flawed share tracking, incorrect reward calculations, and phantom or lost asset distributions, causing financial discrepancies and unfair user outcomes.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-12-plaza-finance-judging/issues/972 

## Found by 
0xmystery, MysteryAuditor, Pablo, X0sauce, bretzel, mxteem, silver\_eth, wellbyt3

## Summary

When an auction starts, the `globalPool` state variable of `BondToken` is updated incorrectly. This leads to the wrong calculation of coupon tokens that bondholders can claim.

## Root Cause

In the [startAuction()](https://github.com/sherlock-audit/2024-12-plaza-finance/blob/14a962c52a8f4731bbe4655a2f6d0d85e144c7c2/plaza-evm/src/Pool.sol#L530-L571) function of `Pool.sol`, the state variable in `bondToken` is updated by calling `bondToken.increaseIndexedAssetPeriod(sharesPerToken)`.

```solidity
  function startAuction() external whenNotPaused() {
  
    ...

    // Calculate the coupon amount to distribute
    uint256 couponAmountToDistribute = (normalizedTotalSupply * normalizedShares)
        .toBaseUnit(maxDecimals * 2 - IERC20(couponToken).safeDecimals());

    auctions[currentPeriod] = Utils.deploy(
      address(new Auction()),
      abi.encodeWithSelector(
        Auction.initialize.selector,
        address(couponToken),
        address(reserveToken),
        couponAmountToDistribute,
        block.timestamp + auctionPeriod,
        1000,
        address(this),
        poolSaleLimit
      )
    );

    // Increase the bond token period
    bondToken.increaseIndexedAssetPeriod(sharesPerToken);

    // Update last distribution time
    lastDistribution = block.timestamp;
  }
```

In the [increaseIndexedAssetPeriod()](https://github.com/sherlock-audit/2024-12-plaza-finance/blob/14a962c52a8f4731bbe4655a2f6d0d85e144c7c2/plaza-evm/src/BondToken.sol#L217-L229) function of `BondToken.sol`, it updates the `globalPool` state variable by pushing a new `PoolAmount` struct to `globalPool.previousPoolAmounts`, setting `sharesPerToken` as `globalPool.sharesPerToken`. Then it updates `globalPool.sharesPerToken` with the new `sharesPerToken`.

This logic is correct only if `sharesPerToken` has not changed. However, the `setSharesPerToken()` function in `Pool.sol` allows for changes to `sharesPerToken`, and `globalPool.sharesPerToken` can only be updated when starting an auction.

If an auction starts with a new `sharesPerToken`, the function uses the previous value (`globalPool.sharesPerToken`), which is outdated. This leads to incorrect calculations of coupon tokens that bondholders can claim.

```solidity
  function increaseIndexedAssetPeriod(uint256 sharesPerToken) public onlyRole(DISTRIBUTOR_ROLE) whenNotPaused() {
    globalPool.previousPoolAmounts.push(
      PoolAmount({
        period: globalPool.currentPeriod,
        amount: totalSupply(),
        sharesPerToken: globalPool.sharesPerToken
      })
    );
    globalPool.currentPeriod++;
    globalPool.sharesPerToken = sharesPerToken;

    emit IncreasedAssetPeriod(globalPool.currentPeriod, sharesPerToken);
  }
```

```solidity
  function setSharesPerToken(uint256 _sharesPerToken) external NotInAuction onlyRole(poolFactory.GOV_ROLE()) {
    sharesPerToken = _sharesPerToken;

    emit SharesPerTokenChanged(sharesPerToken);
  }
```

As a result, the calculation of coupon tokens that bondholders can claim will be based on incorrect values.

```solidity
  function getIndexedUserAmount(address user, uint256 balance, uint256 period) public view returns(uint256) {
    IndexedUserAssets memory userPool = userAssets[user];
    uint256 shares = userPool.indexedAmountShares;

    for (uint256 i = userPool.lastUpdatedPeriod; i < period; i++) {
      shares += (balance * globalPool.previousPoolAmounts[i].sharesPerToken).toBaseUnit(SHARES_DECIMALS);
    }

    return shares;
  }
```

## Internal Pre-Conditions

The state variable `sharesPerToken` of the pool has been modified.

## External Pre-Conditions


## Attack Path


## Impact

The calculation of coupon tokens that bondholders can claim will be incorrect, potentially leading to financial discrepancies.

## Mitigation

Update the `increaseIndexedAssetPeriod()` function to use the current value of `sharesPerToken` instead of `globalPool.sharesPerToken`.

```diff
  function increaseIndexedAssetPeriod(uint256 sharesPerToken) public onlyRole(DISTRIBUTOR_ROLE) whenNotPaused() {
    globalPool.previousPoolAmounts.push(
      PoolAmount({
        period: globalPool.currentPeriod,
        amount: totalSupply(),
-       sharesPerToken: globalPool.sharesPerToken
+       sharesPerToken: sharesPerToken
      })
    );
    globalPool.currentPeriod++;
    globalPool.sharesPerToken = sharesPerToken;

    emit IncreasedAssetPeriod(globalPool.currentPeriod, sharesPerToken);
  }
```


## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/Convexity-Research/plaza-evm/pull/161

---
