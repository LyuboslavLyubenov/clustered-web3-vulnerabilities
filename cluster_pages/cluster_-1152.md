# Cluster -1152

**Rank:** #89  
**Count:** 194  

## Label
**State manipulation via missing validation and improper state updates**  
Attackers exploit unverified state transitions and absent balance checks to inflate balances, override investor records, or lock funds—enabling fund theft, DoS, or unauthorized control through flawed logic and lack of real-time validation.

## Cluster Information
- **Total Findings:** 194

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

**Auto Label:** Insufficient timestamp validation and state consistency lead to unauthorized fund access, premature withdrawals, or permanent fund locking, enabling race conditions and incorrect state transitions.  

**Original Text Preview:**

## Severity: Low Risk

## Context
`StakingManager.sol#L720`

## Description
`withdrawalDelay` is a storage variable used to set the minimum time from which withdrawals can be made by users. The withdrawal process for users includes the following steps:

1. `StakingManager.queueWithdrawal()` (called by the staker).
2. `StakingManager.processL1Operations()` (called by the operator).
3. `StakingManager.withdrawFromSpot()` (called by the operator, after 7 days - the withdrawal time of Hyper-Liquid).
4. `confirmWithdrawal()/batchConfirmWithdrawals()` (called by the staker).

`_processConfirmation()` is called as part of `confirmWithdrawal()/batchConfirmWithdrawals()` and sets the minimum time from which the staker can finalize the withdrawal:

```solidity
function _processConfirmation(address user, uint256 withdrawalId) internal returns (uint256) {
    WithdrawalRequest memory request = _withdrawalRequests[user][withdrawalId];
    // Skip if request doesn't exist or delay period not met
    if (request.hypeAmount == 0 || block.timestamp < request.timestamp + withdrawalDelay) {
        return 0;
    }
    // ...
}
```

In the current version of the code, `withdrawalDelay` can be set to any value as we can see in:

```solidity
function setWithdrawalDelay(uint256 newDelay) external onlyRole(MANAGER_ROLE) {
    withdrawalDelay = newDelay;
    emit WithdrawalDelayUpdated(newDelay);
}
```

The issue here is that if `withdrawalDelay` is less than 7 days, it allows stakers to launch the last step of the withdrawal before their funds are waiting in the contract. In this case, stakers might be able to withdraw funds that belong to other stakers who have requested a withdrawal but have not claimed it yet, causing these users to wait an additional 7 days in the worst case.

## Recommendation
Consider reverting calls to `setWithdrawalDelay()` in case `newDelay` is less than 7 days.

## Acknowledgements
- Kinetiq: Acknowledged.
- Cantina Managed: Acknowledged.

---
### Example 3

**Auto Label:** Failure to validate state before execution enables attackers to trigger reverts, duplicate operations, or infinite loops, leading to denial of service and loss of control.  

**Original Text Preview:**

The [`initRecovery` function](https://github.com/matter-labs/zksync-sso-clave-contracts/blob/c7714c0fe0a33a23acce5aa20355f088d330b4f7/src/validators/GuardianRecoveryValidator.sol#L202-L215) of the `GuardianRecoveryValidator` contract lacks a crucial validation step, allowing for the overwriting of existing recovery data without any checks for ongoing recovery processes. This oversight permits a guardian to initiate a new recovery using incorrect data or to refresh the timestamp, thereby obstructing or postponing the intended recovery process.

This vulnerability is particularly problematic in scenarios where an account is protected by multiple guardians. In such cases, a single guardian acting maliciously can indefinitely disrupt the recovery process by:

1. Overwriting existing recovery data with malicious or incorrect information.
2. Forcing well-intentioned guardians to overwrite this malicious data with the correct information, only for the cycle to repeat ad infinitum.

This cycle not only stalls the recovery process but also leaves no recourse for removing the malicious guardian and regaining control of the account, as the necessary `Transaction` cannot be executed by any party.

To address these vulnerabilities, consider implementing a consensus mechanism among the guardians for recovery processes involving multiple guardians. Such a mechanism would only allow a recovery to proceed once a majority of guardians have concurred on the submitted parameters. Additionally, for accounts with a single guardian, consider enhancing the `initRecovery` function by incorporating a preliminary check to ascertain when a recovery process is still active. These measures will significantly mitigate the risk of overwrites and delays, thereby bolstering the security and efficacy of the recovery process.

***Update:** Partially resolved in [pull request #379](https://github.com/matter-labs/zksync-sso-clave-contracts/pull/379) at commit [f2b6bbf](https://github.com/matter-labs/zksync-sso-clave-contracts/pull/379/commits/f2b6bbfe95f791f157c4174b019b59481f03643e). The Matter Labs team stated:*

> *We decided not to include an N-out-of-M design in this iteration; instead, we added a recommendation to the user-facing documentation to use a Multisig as a guardian to replicate this behavior.*

---
