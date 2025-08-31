# Cluster -1368

**Rank:** #172  
**Count:** 73  

## Label
Rounding and arithmetic flaws in fixed-point calculations enable attackers to exploit precision errors, leading to free token issuance, incorrect allocations, or denial-of-service—undermining fund accuracy and system integrity.

## Cluster Information
- **Total Findings:** 73

## Examples

### Example 1

**Auto Label:** Rounding errors in division operations lead to incorrect asset allocations, causing fund value erosion, user over-receipts, or unintended losses through improper precision handling and lack of consistent rounding direction.  

**Original Text Preview:**

## Severity

Critical Risk

## Description

The `withdraw()` function in `FundContract` converts the requested asset amount into shares using `convertToShares()`, which currently rounds down. However, the ERC4626 standard mandates that `convertToShares()` should round up during withdrawals to prevent users from withdrawing marginally more assets than their shares represent.

Since `withdraw()` uses `convertToShares(assets)` to determine how many shares to burn, rounding down allows users to receive slightly more assets per share than intended. Over multiple withdrawals, this discrepancy compounds, leading to an imbalance between the fund's total assets and its share supply.

## Location of Affected Code

File: [vaults/hyperliquid/fundContract.sol](https://github.com/harmonixfi/core-contracts/blob/f02157aba919dcdd4a1133669361224108c5caef/contracts/vaults/hyperliquid/fundContract.sol)

```solidity
function withdraw(
    uint256 assets,
    address receiver,
    address owner
) public override returns (uint256) {
    // @audit round up should be done instead of round down
    uint256 shares = convertToShares(assets);
    return _withdraw(shares, receiver, owner);
}
```

## Impact

If exploited repeatedly, this rounding error enables users to extract more value than their shares should permit, gradually draining the contract’s assets. This results in financial losses for the protocol and disadvantages other shareholders by diluting the fund’s NAV.

## Recommendation

To fix this issue, `convertToShares()` should round up when used in `withdraw()`, aligning with the ERC4626 standard.

## Team Response

Fixed.

---
### Example 2

**Auto Label:** Rounding and truncation errors in share and token calculations enable attackers to extract profits, drain rewards, or manipulate allocations through imprecise arithmetic and insufficient edge-case validation.  

**Original Text Preview:**

In the `Staking.unstake()` function, the calculation of the shares to be burned for the user is done as follows:

```solidity
79:         uint256 userShare = (userStaked * totalTokenBalance) / info.totalStaked;
80:         require(amount <= userShare, "Cannot unstake more than share");
81:         uint256 fraction = (amount * 1e18) / userShare;
82:
83:         // Decrease user's staked balance proportionally
84:         uint256 sharesToBurn = (userStaked * fraction) / 1e18;
```

The results of the division operations in lines 81 and 84 are rounded down due to the truncation of the division operation, meaning that the shares burned by the user might be underestimated.

Let's consider the following example:

- amount = 1
- userStaked = 10
- totalStaked = 100
- totalTokenBalance = 110
- userShare = 10 \* 110 / 100 = 11
- fraction = 1 \* 1e18 / 11 = 90909090909090909
- sharesToBurn = 10 \* 90909090909090909 / 1e18 = 0

In this case, the user is able to withdraw 1 token, but the shares burned are 0.

The minimum unit of a token is usually a negligible amount, but in the case of tokens with few decimals and high values, the amount can be significant. For example, in WBTC (8 decimals) the minimum unit is valued at approximately 0.001 USD (at current prices) and, the minimum unit of GUSD (2 decimals) is valued at 0.01 USD. In an extreme case where a token with few decimals and high value is used, this attack vector could be exploited to drain the staking pool.

Round up the calculation of `fraction` and `sharesToBurn`, capping the final value to the staked balance of the user.

---
### Example 3

**Auto Label:** Truncation in integer arithmetic causes cumulative precision loss, enabling attackers to evade fees, manipulate debt distribution, and drain funds through systematic rounding errors in interest and share calculations.  

**Original Text Preview:**

**Impact**

This finding summarizes my research on how Truncation could be abused, the goal is to define the mechanisms and quantify the impacts

**Minting of Batch Shares**

Minting (and burning) of batch shares boils down to this line:

https://github.com/liquity/bold/blob/a34960222df5061fa7c0213df5d20626adf3ecc4/contracts/src/TroveManager.sol#L1748-L1749

```solidity
                    batchDebtSharesDelta = currentBatchDebtShares * debtIncrease / _batchDebt;

```

The maximum error here is the ratio of Debt / Shares - 1 meaning that if we rebased the shares to have a 20 Debt per Shares ratio, we can cause at most a rounding that will cause a remainder of 19 debt

This has been explored further in #39  and #40 

**Calculation of Trove Debt**

Boils down to this line:

https://github.com/liquity/bold/blob/a34960222df5061fa7c0213df5d20626adf3ecc4/contracts/src/TroveManager.sol#L932-L933

```solidity
            _latestTroveData.recordedDebt = _latestBatchData.recordedDebt * batchDebtShares / totalDebtShares;

```

The line is distributing the total debt over the total debt shares

The maximum error for this is the % of ownership of `batchDebtShares` over the `totalDebtShares`

Meaning that if we can have very little ownership, we can have a fairly big error

I can imagine the error being used to:
- Skip on paying borrow fees
- Skip on paying management fees

As those 2 operations increase the Debt/Shares

This Python script illustrates the impact 

```python
"""
    The code below demosntrates that:
    - Given a rebased share
    - The maximum amount of debt forgiven for a Trove will be roughly equal to the order of magnitude
        difference between the shares of the Trove and the Shares of the Batch
    
"""

---
