# Cluster -1099

**Rank:** #125  
**Count:** 120  

## Label
Unvalidated transfers and unchecked hook executions enable attackers to manipulate rewards, redirect funds, or seize staked assets through unauthorized state modifications and contract impersonation.

## Cluster Information
- **Total Findings:** 120

## Examples

### Example 1

**Auto Label:** Insufficient state validation and unauthorized state manipulation enable attackers to bypass critical checks, leading to unauthorized transfers, auction manipulation, or protocol devaluation.  

**Original Text Preview:**

The `Folio.bid()` function enforces that bids specify a strict `sellAmount` such that `minSellAmount == sellAmount == maxSellAmount`.

```solidity
function bid(
    //...
    uint256 sellAmount,
    //...
) external nonReentrant notDeprecated sync returns (uint256 boughtAmt) {
    Auction storage auction = auctions[auctionId];

    // checks auction is ongoing and that boughtAmt is below maxBuyAmount
    (, boughtAmt, ) = _getBid(
        --- SNIPPED ---
        sellAmount,             //> minSellAmount
        sellAmount,             //> maxSellAmount
        maxBuyAmount
    );

    --- SNIPPED ---
}
```

The available sell balance is checked at execution time in `RebalanceLib::getBid()`:

```solidity
require(sellAvailable >= params.minSellAmount, IFolio.Folio__InsufficientSellAvailable());
```

This creates a scenario where any change to the protocol balance state between bid creation and execution can cause bids to revert, which can occur through:

1. Other bids: Each successful bid could reduce the `sellToken` balance.
2. Redemptions: These affect both the `sellToken` balance and the `sellLimitBal` calculation, as calculated below:
   ```solidity
    uint256 sellLimitBal = Math.mulDiv(sellLimit, params.totalSupply, D27, Math.Rounding.Ceil);
    uint256 sellAvailable = params.sellBal > sellLimitBal ? params.sellBal - sellLimitBal : 0;
   ```

This issue can be exploited through front-running or can occur naturally through normal protocol usage, blocking bids from executing.
As auction prices decay over time, the protocol could miss favorable execution opportunities.

**Consider the following scenario**:

#### Initial state

0. `totalSupply()` 1000 shares, rebalance targeting reduction from 1000 ETH to 500 ETH, and buy 1,500,000 USDT:

   - 1 BU now consist of 1 ETH 0 USDT.
   - 1 BU traget to consist of 0.5 ETH, 1500 USDT.

1. User A submits a transaction for a full bid: 500 ETH → 1,500,000 USDT at start price (most favorable price).
2. User B front-runs with a minimal bid: 0.0003333 ETH → 1 USDT.
3. The sell token balance decreases to 999.9996667 ETH.
4. The `sellAvailable` becomes 499.9996667 ETH (999.9996667 - 500 ETH).
5. When User A's transaction executes, the check `499.9996667 >= 500` fails.
6. User A's bid reverts, and the protocol loses the opportunity to execute a large bid at a favorable price.

**Recommendation**

Allow a range for `sellAmount` by accepting bids with different `minSellAmount` and `maxSellAmount` values via `Folio.bid()`, allow to fill as much as is available within the range at execution time.

Or, prevent griefing by either freezing `redeem()` during active auctions, or introducing a redemption fee during auctions to disincentivize such attacks.

---
### Example 2

**Auto Label:** Insufficient state validation and unauthorized state manipulation enable attackers to bypass critical checks, leading to unauthorized transfers, auction manipulation, or protocol devaluation.  

**Original Text Preview:**

## Severity: Low Risk

## Context
RedeemController.sol#L30

## Description
The RedeemController enforces only a `minRedemptionAmount` of 1. This opens the door to a spam or queue-stuffing attack: an attacker can enqueue 10,000 minuscule redemption requests, quickly filling the queue (which has a maximum length of 10,000). As a result, legitimate users with valid redemption requests are blocked until the queue is partially cleared with a deposit, causing a temporary denial of service. 

Do note that the attack, even if expensive in terms of gas costs (as it will require 10,000+ external calls), can be performed in a single transaction through the use of a deployed smart contract that interacts directly with the Infinifi protocol.

## Recommendation
Increase `minRedemptionAmount` to a more meaningful threshold so that each redemption request has a non-trivial size. This would discourage attackers from cheaply flooding the queue.

## infiniFi
Acknowledged as a known issue as it is already documented and indicated in the code.

---
### Example 3

**Auto Label:** Insufficient state validation and unauthorized state manipulation enable attackers to bypass critical checks, leading to unauthorized transfers, auction manipulation, or protocol devaluation.  

**Original Text Preview:**

##### Description

In the `ReserveAutomation` contract, the discount (or premium) is computed in the function `currentDiscount()`, as follows:

  

```
    /// @notice Calculates the current discount or premium rate for reserve purchases
    /// @return The current discount as a percentage scaled to 1e18, returns
    /// 1e18 if no discount is applied
    /// @dev Does not apply discount or premium if the sale is not active
    function currentDiscount() public view returns (uint256) {
        if (!isSaleActive()) {
            return SCALAR;
        }

        uint256 decayDelta = startingPremium - maxDiscount;
        uint256 periodStart = getCurrentPeriodStartTime();
        uint256 periodEnd = getCurrentPeriodEndTime();
        uint256 saleTimeRemaining = periodEnd - block.timestamp;

        /// calculate the current premium or discount at the current time based
        /// on the length you are into the current period
        return
            maxDiscount +
            (decayDelta * saleTimeRemaining) /
            (periodEnd - periodStart);
    }
```

  

The value for `periodStart` is obtained from the return of the `getCurrentPeriodEndTime()` function, defined as follows:

```
    /// @notice Returns the end time of the current mini auction period
    /// @return The timestamp when the current mini auction period ends
    /// @dev Returns 0 if no sale is active or if the sale hasn't started yet
    /// @dev Each period is exactly miniAuctionPeriod in length
    function getCurrentPeriodEndTime() public view returns (uint256) {
        uint256 startTime = getCurrentPeriodStartTime();
        if (startTime == 0) {
            return 0;
        }

        return startTime + miniAuctionPeriod - 1;
    }
```

  

In other words, the denominator becomes `periodEnd - periodStart = miniAuctionPeriod - 1`. If the `miniAuctionPeriod` is set to `1` - which is possible, because the condition in the `require` statement of the `initiateSale()` function is as follows: `_auctionPeriod / _miniAuctionPeriod > 1`.

  

In extremely rare conditions, where `_miniAuctionPeriod` is `1` and `_auctionPeriod` is `2`, then the denominator is zero and the contract will revert with a division-by-zero error.

##### BVSS

[AO:A/AC:H/AX:H/R:N/S:C/C:N/A:N/I:M/D:N/Y:N (0.7)](/bvss?q=AO:A/AC:H/AX:H/R:N/S:C/C:N/A:N/I:M/D:N/Y:N)

##### Recommendation

It is recommended to add an explicit check in the `initiateSale()` function to ensure that `_miniAuctionPeriod > 1`. For example:

  

```
require(_miniAuctionPeriod > 1, "ReserveAutomation: Mini auction period too short");
```

  

Alternatively, update the NatSpec documentation to provide proper information regarding the mini auction period.

##### Remediation Comment

**ACKNOWLEDGED:** The **Moonwell team** has acknowledged this issue.

---
