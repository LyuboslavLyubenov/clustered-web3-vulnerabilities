# Cluster -1182

**Rank:** #85  
**Count:** 202  

## Label
Race conditions and improper state validation enable attackers to bypass withdrawal proofs, manipulate balances, or exploit timing to withdraw funds without valid conditions—leading to unauthorized fund access or loss.

## Cluster Information
- **Total Findings:** 202

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

**Auto Label:** Failure to validate critical state invariants during execution flows, enabling race conditions, unchecked approvals, or invalid state transitions that compromise asset integrity and protocol security.  

**Original Text Preview:**

## Severity: Low Risk

## Context
BaseVault.sol#L126-L130

## Description
In the `_executeSubmit()` function, `_noPendingApprovalsInvariant()` is executed before the `_afterSubmitHooks()` call. Therefore, if the after-submit hook makes a callback to the vault to execute `approve()`, that approval will be unchecked. 

This scenario is theoretically possible because the vault does not check if a registered callback has been received during the operation, so the callback can still be executed after the operation. The callback would need to pass the caller and selector checks, so the situation would likely be a configuration error and compromise of the hook or target contract.

## Recommendation
While the above scenario is unlikely to occur, to be on the safe side, consider checking `_noPendingApprovalsInvariant()` after `_afterSubmitHooks()` (i.e., at the end of the `submit()` function) to explicitly guarantee that no approvals are left at the end of the submission.

## Aera
Fixed in PR 218.

## Spearbit
Verified.

---
### Example 3

**Auto Label:** Failure to validate critical state invariants during execution flows, enabling race conditions, unchecked approvals, or invalid state transitions that compromise asset integrity and protocol security.  

**Original Text Preview:**

## Severity: Medium Risk

**Context:** `OracleRegistry.sol#L187-L198`

**Description:**  
The vault owners could accept an unexpected oracle through the `acceptPendingOracle()` function. 

**Consider the following scenario:**

1. The protocol owner schedules an update.
2. The vault owner sees the update and sends a transaction to accept it.
3. Before the transaction is executed, the protocol owner cancels the update and schedules another one.
4. The vault owner ends up accepting an oracle they do not expect.

Eventually, the vault owner will still use the new oracle after the update delay has passed. However, the vault owner may want to take enough time to review the new oracle before accepting it in case of security risks.

**Recommendation:**  
Consider modifying the `acceptPendingOracle()` function so that it takes a parameter `pendingOracle`, which is compared with the on-chain `pendingOracleData[base][quote]` value. If the two values are different, the function should revert.

**Aera:** Fixed in PR 289.  
**Spearbit:** Verified.

---
