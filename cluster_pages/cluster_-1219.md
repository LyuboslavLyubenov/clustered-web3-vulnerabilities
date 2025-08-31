# Cluster -1219

**Rank:** #224  
**Count:** 47  

## Label
Price misrepresentation due to stale or incorrect rate usage in concurrent or dynamic redemption scenarios, enabling user losses, manipulation, or unfair valuation.

## Cluster Information
- **Total Findings:** 47

## Examples

### Example 1

**Auto Label:** Price misrepresentation due to stale or incorrect rate usage in concurrent or dynamic redemption scenarios, enabling user losses, manipulation, or unfair valuation.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/824 

## Found by 
0xgh0st, h2134, jokr, t.aksoy, xiaoming90, zxriptor

### Summary

If CoreRouter is liquidated, some users may suffer more loss than expected.

### Root Cause

When a user redeem tokens from `CoreRouter`,  the underlying token amount to be received is calculated as `user amount * exchange rate`.

[CoreRouter.sol#L117-L118)](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L117-L118):
```solidity
        // Calculate expected underlying tokens
        uint256 expectedUnderlying = (_amount * exchangeRateBefore) / 1e18;
```

The problem is that `CoreRouter` acts as both a supplier and borrower to `LToken`, therefore it's possible that `CoreRouter` is liquidated in `LToken` if its collateral is less than borrowed amount.

If `CoreRouter` is liquidated, it will have less collateral in `LToken` than before, however, such situation is not took into consideration when user s redeem, as a result, the users who redeem earlier can fully redeem, leaving the last users suffer the loss.

### Internal Pre-conditions

NA

### External Pre-conditions

`CoreRouter` is liquidated in `LToken`.

### Attack Path

NA

### Impact

User suffer more loss than expected.

### PoC

_No response_

### Mitigation

When user redeems, the received amount should be calculated based on `CoreRouter`'s collateral in `LToken`.

---
### Example 2

**Auto Label:** Price misrepresentation due to stale or incorrect rate usage in concurrent or dynamic redemption scenarios, enabling user losses, manipulation, or unfair valuation.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/464 

## Found by 
0x15, 0xlucky, 0xzey, 1337web3, Abhan1041, BimBamBuki, Brene, CL001, Etherking, Fade, Frontrunner, HeckerTrieuTien, Hueber, JeRRy0422, Odieli, PNS, Tigerfrake, dimah7, durov, ggg\_ttt\_hhh, ifeco445, ivxylo, kelvinclassic11, m3dython, molaratai, newspacexyz, nodesemesta, odessos42, smbv-1923, t.aksoy, wickie, x0rc1ph3r, zxriptor

### Summary

The `redeem` [function](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L100) in `CoreRouter.sol` (lines 82-110) is responsible for redeeming a user's LTokens for the underlying asset. The process involves:
1.  Fetching the current exchange rate using `LTokenInterface(_lToken).exchangeRateStored()` (line 93). This rate can be stale if, for example, interest has accrued within the `LToken` but its `accrueInterest()` function hasn't been recently called to update the stored rate.
2.  Calculating `expectedUnderlying` tokens to be returned to the user based on this potentially stale rate (line 96: `uint256 expectedUnderlying = (_amount * exchangeRateBefore) / 1e18;`).
3.  Calling `LErc20Interface(_lToken).redeem(_amount)` (line 98). This function call is expected to result in the `LToken` contract transferring the actual amount of underlying tokens (based on its *own* current, possibly updated, exchange rate and any applicable fees) to the `CoreRouter` contract.
4.  Transferring the pre-calculated `expectedUnderlying` amount from `CoreRouter` to the `msg.sender` (the user) (line 99: `IERC20(_token).transfer(msg.sender, expectedUnderlying);`).

The vulnerability arises because `CoreRouter` transfers `expectedUnderlying` without confirming the `actualUnderlyingReceived` from the `LToken.redeem()` call.
*   If `actualUnderlyingReceived < expectedUnderlying` (e.g., `exchangeRateStored()` was erroneously high, or the `LToken` applies redemption fees not reflected in `exchangeRateStored()`), `CoreRouter` pays out more than it received, leading to a drain of its underlying token reserves for that market.
*   If `actualUnderlyingReceived > expectedUnderlying` (e.g., `exchangeRateStored()` was stale and low, and the `LToken`'s internal rate was higher after interest accrual during its `redeem` process), `CoreRouter` pays out less than it received, causing the excess funds to be trapped in the `CoreRouter` contract.

### Root Cause

The `CoreRouter.redeem()` function trusts the `expectedUnderlying` amount, calculated using `LTokenInterface(_lToken).exchangeRateStored()` *before* the `LToken`'s redeem operation, for the final payout to the user. It does not verify or use the actual amount of underlying tokens it receives from the `LErc20Interface(_lToken).redeem()` call. The `LToken` itself may use a more current (updated) exchange rate or apply fees during its `redeem` execution, leading to a mismatch between `expectedUnderlying` and the actual tokens received by `CoreRouter`.

### Internal Pre-conditions

*   A user calls the `redeem(uint256 _amount, address payable _lToken)` function.
*   The `_lToken` provided is a valid LToken recognized by `lendStorage`.
*   The user has a sufficient balance of `_lToken` (tracked by `lendStorage.totalInvestment`).
*   The user's redemption request passes liquidity checks (`lendStorage.getHypotheticalAccountLiquidityCollateral`).
*   `CoreRouter` potentially holds some reserve of the underlying token to cover discrepancies if it overpays.

### External Pre-conditions

*   A discrepancy exists between the value returned by `LTokenInterface(_lToken).exchangeRateStored()` (when called by `CoreRouter` at line 93) and the effective exchange rate or net amount that `LErc20Interface(_lToken).redeem()` uses/provides when transferring underlying tokens to `CoreRouter`. This can occur if:
    *   `exchangeRateStored()` is stale (has not been updated recently via `accrueInterest()` in the `LToken`).
    *   The `LToken`'s `redeem()` function internally calls `accrueInterest()`, changing the exchange rate *after* `CoreRouter` fetched `exchangeRateStored()`.
    *   The `LToken`'s `redeem()` function applies fees that reduce the net underlying tokens transferred to `CoreRouter`, and these fees are not reflected in the `exchangeRateStored()` value.

### Attack Path

**Scenario 1: Fund Drain from CoreRouter (CoreRouter overpays user)**
1.  A situation occurs where `LTokenInterface(_lToken).exchangeRateStored()` provides a rate (`R_stale`) that will lead to an `expectedUnderlying` calculation higher than what `CoreRouter` will actually receive from `LToken.redeem()`. This could be because:
    *   The `LToken` applies a redemption fee (e.g., 0.1% of the redeemed amount) which is not factored into `R_stale`.
    *   `R_stale` is anomalously high compared to the rate the `LToken` will use internally (e.g., `R_stale` is from a moment before a downward rate correction within the `LToken`).
2.  A user (or an entity aware of this discrepancy) calls `CoreRouter.redeem(_amount, _lToken)`.
3.  `CoreRouter` calculates `expectedUnderlying = (_amount * R_stale) / 1e18`.
4.  `CoreRouter` calls `LErc20Interface(_lToken).redeem(_amount)`. The `LToken` contract processes this, and due to fees or a lower internal rate, transfers `actualUnderlyingReceived` to `CoreRouter`, where `actualUnderlyingReceived < expectedUnderlying`.
5.  `CoreRouter` transfers `expectedUnderlying` to the user (line 99).
6.  `CoreRouter` has now paid out `expectedUnderlying - actualUnderlyingReceived` more of the underlying token than it received for this specific operation.
7.  Repeated redemptions under these conditions will continuously drain `CoreRouter`'s reserves of that underlying token.

**Scenario 2: Funds Trapped in CoreRouter (User receives less than LToken's current value)**
1.  `LTokenInterface(_lToken).exchangeRateStored()` provides a rate (`R_stale`) which is lower than the `LToken`'s actual current exchange rate (e.g., interest has accrued in the `LToken`, but `R_stale` hasn't been updated).
2.  A user calls `CoreRouter.redeem(_amount, _lToken)`.
3.  `CoreRouter` calculates `expectedUnderlying = (_amount * R_stale) / 1e18`.
4.  `CoreRouter` calls `LErc20Interface(_lToken).redeem(_amount)`. The `LToken` contract processes this (potentially calling `accrueInterest()` internally), uses its up-to-date higher exchange rate, and transfers `actualUnderlyingReceived` to `CoreRouter`, where `actualUnderlyingReceived > expectedUnderlying`.
5.  `CoreRouter` transfers `expectedUnderlying` (the lower amount) to the user.
6.  The difference, `actualUnderlyingReceived - expectedUnderlying`, remains trapped in the `CoreRouter` contract, as it's not accounted for.

### Impact

1.  **Gradual Drain of Underlying Token Reserves:** If `CoreRouter` consistently receives fewer underlying tokens from `LToken.redeem()` than its `expectedUnderlying` calculation (e.g., due to LToken redemption fees), it will transfer out more tokens than it receives. This leads to a gradual depletion of `CoreRouter`'s reserves for that specific underlying token. If these reserves are exhausted, further operations for that market (like other users' redemptions) could fail or be impaired, potentially damaging the protocol's solvency for that market.
2.  **Trapped Funds in CoreRouter:** If `CoreRouter` receives more underlying tokens than `expectedUnderlying` (e.g., due to a stale, lower `exchangeRateStored()`), the excess tokens become trapped in the `CoreRouter` contract. While not a direct loss to the protocol's overall holdings, these funds are not correctly allocated (e.g., to the redeeming user who might have been entitled to more based on the LToken's actual current rate) and are not managed or recoverable by any apparent mechanism. This represents an accounting discrepancy and a loss of value for users or the protocol's distributable surplus.

### PoC

_No response_

### Mitigation

_No response_

---
### Example 3

**Auto Label:** **Incorrect price or rate calculation due to flawed mathematical logic or stale data, enabling manipulation, arbitrage, or value leakage.**  

**Original Text Preview:**

## Summary

The `StabilityPool::getExchangeRate()` function is hardcoded to return a fixed value of `1e18`, which assumes a static 1:1 exchange rate between `rToken and deToken`. This implementation does not account for real-time changes in token supply and demand. As a result, users may receive incorrect amounts when depositing or redeeming tokens, leading to potential financial imbalances and arbitrage exploits.

## Vulnerability Details

```Solidity
function getExchangeRate() public view returns (uint256) {
    // uint256 totalDeCRVUSD = deToken.totalSupply();
    // uint256 totalRcrvUSD = rToken.balanceOf(address(this));
    // if (totalDeCRVUSD == 0 || totalRcrvUSD == 0) return 10**18;

    // uint256 scalingFactor = 10**(18 + deTokenDecimals - rTokenDecimals);
    // return (totalRcrvUSD * scalingFactor) / totalDeCRVUSD;
    return 1e18;
}
```

### **Issue:**

1. **Static Exchange Rate:**

   * The function always returns `1e18`, meaning the exchange rate between `rToken` and `deToken` is assumed to be fixed at 1:1.
   * The commented-out code suggests an original intention to make the rate dynamic based on token supply but was removed.

2. **Incorrect Deposit and Redemption Calculations:**

   * The deposit function (`deposit()`) relies on `calculateDeCRVUSDAmount()`, which uses `getExchangeRate()`.
   * The redemption function (`calculateRcrvUSDAmount()`) also uses `getExchangeRate()`.
   * Since `getExchangeRate()` is hardcoded, these calculations may not reflect real market conditions.

3. **Arbitrage Risks:**

   * If the real market value of `rToken` fluctuates, users can **deposit undervalued tokens and redeem overvalued ones**, extracting unearned profits.
   * This could result in **protocol insolvency** if redemptions exceed available reserves.

4. **Liquidity Imbalance:**

   * The total supply of `deToken` may become misaligned with the actual available `rToken` balance.
   * If too many deposits occur before an update, redemptions may fail due to insufficient reserves.



## Impact

<https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/pools/StabilityPool/StabilityPool.sol#L211-L212>

**Real-World Impact** A similar issue occurred in **DeFi lending protocols**, where mispriced exchange rates led to excessive borrowing and redemption exploits. A well-known case was the **Iron Finance Bank Run**, where an artificially pegged stablecoin lost parity, causing mass redemptions and liquidity depletion.

## Tools Used

## Recommendations

Replace the hardcoded `1e18` with a formula that correctly derives the rate based on `deToken.totalSupply()` and `rToken.balanceOf(address(this))`.

---
