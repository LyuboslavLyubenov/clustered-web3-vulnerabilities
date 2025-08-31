# Cluster -1007

**Rank:** #33  
**Count:** 585  

## Label
Failure to validate or check contract state before execution leads to incorrect state updates, unauthorized operations, or silent failures, risking asset loss, inconsistent accounting, or exploitation during invalid or transitional states.

## Cluster Information
- **Total Findings:** 585

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

**Auto Label:** Missing state validation and precondition checks lead to inconsistent, unresolvable, or exploitable contract states during critical operations like pausing, unpausing, or state transitions.  

**Original Text Preview:**

## Severity

**Impact:** High - Allows unauthorized withdrawal of assets during critical situations when the vault is paused.

**Likelihood:** Low - This issue occurs only when the vault is in a paused state.

## Description

The vault's deposit, mint, and withdraw functionalities are halted when it is paused. This is implemented through the `whenNotPaused` modifier in the following functions:

        function deposit(
            uint256 _amount,
            address _receiver
        ) public whenNotPaused returns (uint256 shares) {

        function safeDeposit(
            uint256 _amount,
            uint256 _minShareAmount,
            address _receiver
        ) public whenNotPaused returns (uint256 shares) {

        function withdraw(
            uint256 _amount,
            address _receiver,
            address _owner
        ) external whenNotPaused returns (uint256) {

        function safeWithdraw(
            uint256 _amount,
            uint256 _minAmount,
            address _receiver,
            address _owner
        ) public whenNotPaused returns (uint256 amount) {

However, the redeem function is not pausable because the `whenNotPaused` modifier is not applied . This absence allows users to withdraw assets from the vaul when they should not.

        function redeem(
            uint256 _shares,
            address _receiver,
            address _owner
        ) external returns (uint256 assets) {

        function safeRedeem(
            uint256 _shares,
            uint256 _minAmountOut,
            address _receiver,
            address _owner
        ) external returns (uint256 assets) {

## Recommendations

Adding the whenNotPaused modifier to both the redeem and safeRedeem functions.

```diff
    function redeem(
        uint256 _shares,
        address _receiver,
        address _owner
-   ) external returns (uint256 assets) {
+   ) external whenNotPaused returns (uint256 assets) {
        return _withdraw(previewRedeem(_shares), _shares, _receiver, _owner);
    }

    function safeRedeem(
        uint256 _shares,
        uint256 _minAmountOut,
        address _receiver,
        address _owner
-    ) external returns (uint256 assets) {
+    ) external whenNotPaused returns (uint256 assets) {
        assets = _withdraw(
            previewRedeem(_shares),
            _shares, // _shares
            _receiver, // _receiver
            _owner // _owner
        );
        if (assets < _minAmountOut) revert AmountTooLow(assets);
    }
```

---
### Example 3

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
