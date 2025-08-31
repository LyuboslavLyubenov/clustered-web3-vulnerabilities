# Cluster -1020

**Rank:** #147  
**Count:** 95  

## Label
Inadequate price adjustment mechanisms lead to misaligned token valuations, distorted incentives, and exploitable market behaviors due to lack of downward rebase capability or discontinuous rebase events.

## Cluster Information
- **Total Findings:** 95

## Examples

### Example 1

**Auto Label:** Inadequate price adjustment mechanisms lead to misaligned token valuations, distorted incentives, and exploitable market behaviors due to lack of downward rebase capability or discontinuous rebase events.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Medium

## Description

The function `rebase()` has been used to increase the share price and apply staking rewards. The issue is that staking has its own risks and stake amounts can be slashed (In Commume or Bittensor network) and current logic doesn't support decreasing the share price. Also, some tokens can be stolen or lost when bridging so it would be necessary to have the ability to decrease the share price to adjust the share price according to those events.

## Recommendations

Add the ability to decrease the share price too.

---
### Example 2

**Auto Label:** Misaligned token emissions due to flawed rebase calculations, leading to inflated supply and broken economic assumptions.  

**Original Text Preview:**

According to the doc(https://docs.kittenswap.finance/tokenomics/initial-distribution), our initial emission will start from `20,000,000 KITTEN (2%)`.

When we check our implementation, the actual initial amount is `20,000,000 + 20,000,000 * 30% + (20,000,000 * 130% * 5%)`. This will be larger than the expected emission amount in our doc.

```solidity
    function updatePeriod() external returns (bool) {
        uint256 currentPeriod = getCurrentPeriod();
        if (currentPeriod > lastMintedPeriod) {
            uint256 emissions = nextEmissions;
            nextEmissions = calculateNextEmissions(emissions);
            // 2% * circulating supply
            uint256 _tailEmissions = tailEmissions();
            uint256 rebase = calculateRebase(emissions);
            // emssion + emission * 30%
            uint256 treasuryEmissions = ((emissions + rebase) * treasuryRate) /
                PRECISION;
            uint256 mintAmount = emissions + rebase + treasuryEmissions;
            // kitten token in this minter.
            uint256 kittenBal = kitten.balanceOf(address(this));
            if (kittenBal < mintAmount) {
                kitten.mint(address(this), mintAmount - kittenBal);
            }
        }
    }
```

Recommendation: Consistent between the doc and the implementation.

---
### Example 3

**Auto Label:** Misaligned token emissions due to flawed rebase calculations, leading to inflated supply and broken economic assumptions.  

**Original Text Preview:**

According to the doc(https://docs.kittenswap.finance/tokenomics/initial-distribution), our initial emission will start from `20,000,000 KITTEN (2%)`.

When we check our implementation, the actual initial amount is `20,000,000 + 20,000,000 * 30% + (20,000,000 * 130% * 5%)`. This will be larger than the expected emission amount in our doc.

```solidity
    function updatePeriod() external returns (bool) {
        uint256 currentPeriod = getCurrentPeriod();
        if (currentPeriod > lastMintedPeriod) {
            uint256 emissions = nextEmissions;
            nextEmissions = calculateNextEmissions(emissions);
            // 2% * circulating supply
            uint256 _tailEmissions = tailEmissions();
            uint256 rebase = calculateRebase(emissions);
            // emssion + emission * 30%
            uint256 treasuryEmissions = ((emissions + rebase) * treasuryRate) /
                PRECISION;
            uint256 mintAmount = emissions + rebase + treasuryEmissions;
            // kitten token in this minter.
            uint256 kittenBal = kitten.balanceOf(address(this));
            if (kittenBal < mintAmount) {
                kitten.mint(address(this), mintAmount - kittenBal);
            }
        }
    }
```

Recommendation: Consistent between the doc and the implementation.

---
