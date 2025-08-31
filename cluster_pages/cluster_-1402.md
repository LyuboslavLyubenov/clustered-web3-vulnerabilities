# Cluster -1402

**Rank:** #181  
**Count:** 67  

## Label
Inadequate input validation and logic errors lead to incorrect state updates, invalid position creation, or improper liquidity calculations, causing financial inaccuracies, transaction reverts, or exploitable discrepancies in user balances and pool behavior.

## Cluster Information
- **Total Findings:** 67

## Examples

### Example 1

**Auto Label:** Inadequate input validation and logic errors lead to incorrect state updates, invalid position creation, or improper liquidity calculations, causing financial inaccuracies, transaction reverts, or exploitable discrepancies in user balances and pool behavior.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** High

## Description

The protocol implements a staking system where multiple `StakingManager` contracts can coexist, sharing profits and losses through a common `ValidatorManager`. Each `StakingManager` maintains its own accounting of staked and claimed amounts via `totalStaked` and `totalClaimed` state variables, while sharing rewards and slashing through the `ValidatorManager`.

The critical issue lies in the `getExchangeRatio()` function, which calculates the exchange rate between HYPE and kHYPE tokens. The current implementation uses local accounting values but global PnL figures, leading to incorrect exchange rate calculations across different `StakingManager` instances.

The exchange rate calculation follows this formula:

```solidity
exchangeRate = (totalStaked + totalRewards - totalClaimed - totalSlashing) / kHYPESupply
```

Where:

- `totalStaked` and `totalClaimed` are local to each `StakingManager`
- `totalRewards` and `totalSlashing` are global values from `ValidatorManager`
- `kHYPESupply` is the total supply of kHYPE tokens

This creates a significant discrepancy as each `StakingManager` will calculate different exchange rates based on its local stake/claim values, while sharing the same rewards and slashing. 

### Proof of Concept

1. Two `StakingManager` contracts (SM1 and SM2) are deployed

2. User A stakes 100 HYPE through SM1
    - SM1.totalStaked = 100
    - SM2.totalStaked = 0

3. User B stakes 100 HYPE through SM2
    - SM1.totalStaked = 100
    - SM2.totalStaked = 100

4. `ValidatorManager` records 20 HYPE in rewards

5. Exchange rate calculation:
    - SM1: (100 + 20 - 0 - 0) / 100 = 1.2
    - SM2: (100 + 20 - 0 - 0) / 100 = 1.2

The rates appear equal but are incorrect because each manager only sees half of the total staked amount.

6. Correct rate should be: (200 + 20 - 0 - 0) / 200 = 1.1

The current design creates arbitrage opportunities and incorrect valuation of kHYPE tokens depending on which `StakingManager` users interact with.

## Recommendations

The `ValidatorManager` should track global staking totals, and `StakingManager` instances should query these global values for exchange rate calculations.

---
### Example 2

**Auto Label:** Inadequate input validation and logic errors lead to incorrect state updates, invalid position creation, or improper liquidity calculations, causing financial inaccuracies, transaction reverts, or exploitable discrepancies in user balances and pool behavior.  

**Original Text Preview:**

The protocol potentially has inconsistent token transfer for `pnl` payouts between `OperationalTreasury` and `Pool` contracts. While `Pool` uses the position's `buyToken` for payouts, `OperationalTreasury` uses the current `baseSetUp.token`. If `OperationalTreasury` is updated and reinitialized the base setup with a different base token, users will receive payouts in the wrong token.

```solidity
function payOff(uint256 positionID, address account) external override nonReentrant whenNotPaused {
    --- SNIPPED ---
    if (pnl > 0) baseSetUp.token.safeTransfer(account, netAmount);
    emit Paid(positionID, account, netAmount, feesUSD);
}
```

```solidity
function _payout(Position memory pos, PositionClose memory close) internal returns (uint256 pnl) {
    --- SNIPPED ---
    strg.ledger.state.poolAmount[pos.buyToken] -= pnl;
    _doTransferOut(pos.buyToken, msg.sender /* operational treasury*/, pnl);
}
```

Recommendation:

Record the original `baseSetUp.token` or `buyToken` with each position in the `OperationalTreasury` and use it for payouts.

---
### Example 3

**Auto Label:** Inadequate input validation and arithmetic safeguards lead to incorrect state transitions, financial misalignment, and potential overflow or underflow, enabling malicious actors to manipulate fees, liquidity, or ratios.  

**Original Text Preview:**

## Diﬃculty: Low

## Type: Data Validation

## Description
The pool contract’s reserve ratio validation logic incorrectly uses a logical OR operator instead of AND, allowing transactions to proceed even when the reserve ratio is outside the specified range. The validation should consider a ratio valid only when it is both above the minimum **AND** below the maximum, but the current implementation considers it valid if it is either above the minimum **OR** below the maximum.

The identified vulnerability resides in the `deposit_liquidity` function, as illustrated in figure 2.1, and in the `withdraw_liquidity` function.

```c
int valid = (compare_fractions(
    min_nominator,
    denominator,
    storage::reserve1,
    storage::reserve2
) <= 0) | (compare_fractions(
    max_nominator,
    denominator,
    storage::reserve1,
    storage::reserve2
) >= 0);
ifnot (valid) {
    excno = errors::reserves_ratio_failed;
}
```

**Figure 2.1:** The incorrect logical OR operator in the validation logic (dex/contracts/pool/include/jetton_based.fc#237–250)

The same logical operator error in the reserve ratio validation logic has also been found in the fee validation mechanism. However, in this case, the impact is lower because only admin operations can trigger the vulnerability, rather than arbitrary user transactions.

## Exploit Scenario
Alice submits a liquidity deposit transaction with specific min/max ratio parameters. Eve executes a swap that dramatically alters the pool’s ratio to be outside Alice’s intended range. Due to the logical OR error, Alice’s transaction still executes even though the ratio is now completely unfavorable.

## Recommendations
- **Short term:** Replace the logical OR operator (`|`) with a logical AND operator (`&`) in the reserve ratio validation logic to ensure that both conditions must be met for a valid range check.

- **Long term:** Implement comprehensive unit tests for the reserve ratio validation logic with various edge cases, including values at and beyond the boundaries of the specified ranges.

---
