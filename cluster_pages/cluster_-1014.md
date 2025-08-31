# Cluster -1014

**Rank:** #37  
**Count:** 564  

## Label
Lack of proper event emissions and view functions undermines transparency, auditability, and real-time observability, enabling misrepresentation of state and limiting external access to critical on-chain data.

## Cluster Information
- **Total Findings:** 564

## Examples

### Example 1

**Auto Label:** Missing event emissions in state-changing operations compromise transparency, auditability, and real-time observability, enabling undetected malicious activity and undermining trust in decentralized systems.  

**Original Text Preview:**

##### Description
`LimitOrderManager.setTickSpacing()` changes the `tickSpacings` mapping but emits no event, leaving off-chain indexers unaware of configuration changes.
<br/>
##### Recommendation
We recommend emitting an event whenever tick-spacing is updated.

> **Client's Commentary:**
> Commit: https://github.com/cryptoalgebra/plugins-monorepo/commit/e877bc129c4454326a3de565c9554587a256449e

---

---
### Example 2

**Auto Label:** Missing event emissions in state-changing operations compromise transparency, auditability, and real-time observability, enabling undetected malicious activity and undermining trust in decentralized systems.  

**Original Text Preview:**

## Diﬃculty: Medium

## Type: Data Validation

## Description
Several event-emitting Folio functions can be called repeatedly, without intervening calls to related event-emitting functions. Off-chain code expecting to process multiple kinds of events could behave incorrectly when seeing just one type of event.

The following functions in question are affected:
- `startRebalance` (which emits `RebalanceStarted`) can be called repeatedly without intervening calls to `endRebalance` (which emits `RebalanceEnded`).
- `endRebalance` (which emits `RebalanceEnded`) can be called repeatedly without intervening calls to `startRebalance` (which emits `RebalanceStarted`).
- `closeAuction` (which emits `AuctionClosed`) can be called repeatedly without intervening calls to `openAuction` (which emits `AuctionOpened`).

Note that `openAuction` does not have this problem. That is, multiple calls to `openAuction` require intervening calls to `closeAuction` or `endRebalance`.

Also, note that some of the above functions have side effects besides emitting events. For example, repeated calls to `endRebalance` will result in repeated changes to `rebalance.availableUntil` (figure 2.1), though we could find no way that this would cause harm:

```solidity
794    function endRebalance() external nonReentrant {
...
802        emit RebalanceEnded(rebalance.nonce);
803
804        // do not revert, to prevent griefing
805        rebalance.nonce++; // advancing nonce clears auctionEnds
806        rebalance.availableUntil = block.timestamp;
807    }
```
*Figure 2.1: Excerpt of the definition of `endRebalance` (reserve-index-dtf/contracts/Folio.sol#794–807)*

Reserve explained that some of the behavior described in this finding is intentional. For example, suppose an auction launcher calls `endRebalance` on a Folio. If a governance proposal calls `endRebalance` on the same Folio, the proposal’s call should not revert because of the auction launcher’s call.

## Exploit Scenario
Alice writes a bot that trades against Reserve Folios. Alice’s bot expects auction sequences to be bracketed by `RebalanceStarted` and `RebalanceEnded` events, and for each auction to begin with an `AuctionOpened` event. Bo maintains one of the Folios that Alice’s bot trades against. Bob calls `startRebalance`, `endRebalance`, or `closeAuction` multiple times (for unknown reasons). Bob’s transactions succeed, but the emitted events cause Alice’s bot to behave incorrectly. Alice suffers financial loss as a result.

## Recommendations
**Short term:** Adjust `startRebalance` so that one or more consecutive calls to that function emit exactly one `RebalanceStarted` event. Adjust `endRebalance` and `closeAuction` similarly. Eliminate any other undesirable side effects resulting from repeated calls to these functions. Taking these steps will produce more predictable event sequences, reducing the likelihood of off-chain code behaving incorrectly.

**Long term:** Incorporate fuzzing with Medusa into your workflow. Medusa can generate arbitrary call sequences, which could help to expose problems like the one described here.

---
### Example 3

**Auto Label:** Missing event emissions in state-changing operations compromise transparency, auditability, and real-time observability, enabling undetected malicious activity and undermining trust in decentralized systems.  

**Original Text Preview:**

**Description:** [`LINKMigrator::setQueueDepositMin`](https://github.com/stakedotlink/contracts/blob/0bd5e1eecd866b2077d6887e922c4c5940a6b452/contracts/linkStaking/LINKMigrator.sol#L119-L125) and [`PriorityPool::setQueueBypassController`](https://github.com/stakedotlink/contracts/blob/0bd5e1eecd866b2077d6887e922c4c5940a6b452/contracts/core/priorityPool/PriorityPool.sol#L678-L685) change internal state without emitting events. Events are important for off-chain tracking and transparency. Consider emitting events from these functions.

**stake.link:**
Acknowledged.

---
