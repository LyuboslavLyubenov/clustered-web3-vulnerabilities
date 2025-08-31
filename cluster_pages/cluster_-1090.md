# Cluster -1090

**Rank:** #397  
**Count:** 12  

## Label
Redundant or flawed state checks and recalculations lead to inconsistent state, unexpected behavior, or denial-of-service, compromising reliability and user access.

## Cluster Information
- **Total Findings:** 12

## Examples

### Example 1

**Auto Label:** Redundant or flawed state checks and recalculations lead to inconsistent state, unexpected behavior, or denial-of-service, compromising reliability and user access.  

**Original Text Preview:**

##### Description
In the `DIAExternalStaking` contract, the daily withdrawal limit calculation in the `checkDailyWithdrawalLimit` modifier is tied to the `totalPoolSize`. The issue arises because:

1. The daily withdrawal limit is calculated as:
```solidity
availableDailyLimit = (totalPoolSize * withdrawalCapBps) / 10000;
```
2. When `totalPoolSize` decreases (due to unstaking), the `availableDailyLimit` also decreases proportionally, even if the remaining pool size is still significant.
3. This creates a situation where:
- A user may have successfully withdrawn a certain amount earlier in the day
- Later withdrawals may be blocked due to the reduced `availableDailyLimit`
- Even if the total pool size is still substantial, the withdrawal limit may be too low to allow further withdrawals

Users may be unable to withdraw their funds even though the total pool size is still significant and it creates inconsistent withdrawal behavior throughout the day.
<br/>
##### Recommendation
We recommend calculating the daily limit based on the initial pool size at the start of the day:
```solidity
if (block.timestamp / SECONDS_IN_A_DAY > lastWithdrawalResetDay) {
    totalDailyWithdrawals = 0;
    lastWithdrawalResetDay = block.timestamp / SECONDS_IN_A_DAY;
    availableDailyLimit = (totalPoolSize * withdrawalCapBps) / 10000;
}
```

`availableDailyLimit` variable may be stored in the storage and updated every time the new day starts.

> **Client's Commentary:**
> This is by design and the limit decreases until it reaches the minimal cap, which is the not limited. As part of the review for this issue we noticed that in the the unstaking logic the amount selection should come at the request stage, not unstake, therefore it was moved to a step earlier.

---
### Example 2

**Auto Label:** Redundant or unguarded state updates due to missing conditionals, leading to inconsistent state or exploitable race conditions in critical execution paths.  

**Original Text Preview:**

##### Description

The `burn` function contains a redundant call to `validate_pool_id`:

<https://github.com/mira-amm/mira-v1-core/blob/e0afbeb56ba8c01e223f2552979ee55e461d24a9/contracts/mira_amm_contract/src/main.sw#L513>

```
fn burn(pool_id: PoolId, to: Identity) -> (u64, u64) {
    reentrancy_guard();
    validate_pool_id(pool_id);

    ...
}
```

as it is already done when calling `get_pool`:

<https://github.com/mira-amm/mira-v1-core/blob/e0afbeb56ba8c01e223f2552979ee55e461d24a9/contracts/mira_amm_contract/src/main.sw#L518>

```
    fn burn(pool_id: PoolId, to: Identity) -> (u64, u64) {
        reentrancy_guard();
        validate_pool_id(pool_id);

        ...

        let mut pool = get_pool(pool_id);

        ...
```

```
fn get_pool(pool_id: PoolId) -> PoolInfo {
    let pool = get_pool_option(pool_id);
    require(pool.is_some(), InputError::PoolDoesNotExist(pool_id));
    pool.unwrap()
}
```

```
fn get_pool_option(pool_id: PoolId) -> Option<PoolInfo> {
    validate_pool_id(pool_id);
    storage.pools.get(pool_id).try_read()
}
```

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:N/S:C (0.0)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:N/S:C)

##### Recommendation

Remove the first call to `validate_pool_id`.

##### Remediation

**SOLVED:** The **Mira team** solved this issue.

##### Remediation Hash

6d4d5efcf29ce00d931bd70d14df57e867f39331

##### References

[mira-amm/mira-v1-core/contracts/mira\_amm\_contract/src/main.sw#L511](https://github.com/mira-amm/mira-v1-core/blob/main/contracts/mira_amm_contract/src/main.sw#L511)

---
### Example 3

**Auto Label:** Redundant or flawed state checks and recalculations lead to inconsistent state, unexpected behavior, or denial-of-service, compromising reliability and user access.  

**Original Text Preview:**

## Description
In the LRTDepositPool contract, the function `removeManyNodeDelegatorContractsFromQueue()` on line [349] iteratively calls `removeNodeDelegatorContractFromQueue()`, invoking the `onlyLRTAdmin` modifier with each iteration. This repeated check for administrative privileges introduces unnecessary gas costs, as the modifier’s condition is unlikely to change between iterations within a single transaction.

## Recommendations
Consider optimizing gas usage by restructuring the code to perform administrative privilege checks only once per transaction. One approach could involve consolidating the modifier check outside of the loop or redesigning the removal logic to reduce redundant checks.

## Resolution
The development team has fixed the above issue in commit f98011b.

---
