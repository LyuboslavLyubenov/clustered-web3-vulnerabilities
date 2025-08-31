# Cluster -1202

**Rank:** #381  
**Count:** 14  

## Label
Redundant storage usage leading to increased gas costs, inconsistent state, and poor performance due to unnecessary variable declarations, improper memory-to-storage interactions, and inefficient storage reads.

## Cluster Information
- **Total Findings:** 14

## Examples

### Example 1

**Auto Label:** Inefficient storage layout leading to redundant reads/writes and increased gas costs due to scattered state variables, with no direct security impact but higher operational overhead.  

**Original Text Preview:**

##### Description
The issue is identified within the [HanjiLOB](https://github.com/longgammalabs/hanji-contracts/blob/70b15ec4d9e7578248141604503843716a67d875/src/HanjiLOB.sol#L122) contract.

The storage variables `marketmaker` and `should_invoke_on_trade` are frequently accessed together. Gas usage can be optimized by placing these two variables in the same storage slot, reducing the number of storage read and write operations:

```solidity!
struct MarketmakerConfig {
    address marketmaker;
    bool should_invoke_on_trade;
}
```

Additionally, the bytecode and code readability can be improved by moving the check and the call to `ITradeConsumer(marketmaker).onTrade(isAsk)` into a separate internal function, thereby reducing the code dublication.

This issue is classified as **low** severity because, while it does not pose a security risk, it can lead to gas savings, especially in contracts that perform frequent operations.
##### Recommendation
We recommend reworking the marketmaker callbacks as described above by grouping the related variables in a struct and refactoring the callback logic into a separate internal function.

---
### Example 2

**Auto Label:** Inefficient storage layout leading to redundant reads/writes and increased gas costs due to scattered state variables, with no direct security impact but higher operational overhead.  

**Original Text Preview:**

##### Description
The issue is identified within the [HanjiLOB](https://github.com/longgammalabs/hanji-contracts/blob/70b15ec4d9e7578248141604503843716a67d875/src/HanjiLOB.sol#L122) contract.

The storage variables `marketmaker` and `should_invoke_on_trade` are frequently accessed together. Gas usage can be optimized by placing these two variables in the same storage slot, reducing the number of storage read and write operations:

```solidity!
struct MarketmakerConfig {
    address marketmaker;
    bool should_invoke_on_trade;
}
```

Additionally, the bytecode and code readability can be improved by moving the check and the call to `ITradeConsumer(marketmaker).onTrade(isAsk)` into a separate internal function, thereby reducing the code dublication.

This issue is classified as **low** severity because, while it does not pose a security risk, it can lead to gas savings, especially in contracts that perform frequent operations.
##### Recommendation
We recommend reworking the marketmaker callbacks as described above by grouping the related variables in a struct and refactoring the callback logic into a separate internal function.

---
### Example 3

**Auto Label:** Inefficient storage layout leading to redundant reads/writes and increased gas costs due to scattered state variables, with no direct security impact but higher operational overhead.  

**Original Text Preview:**

##### Description
The [`transferCommissions`](https://github.com/longgammalabs/hanji-contracts/blob/09b6188e028650b9c1758010846080c5f8c80f8e/src/OnchainLOB.sol#L582) function updates commissions by modifying the `Trader` struct for both the administrator and the market maker. A more efficient approach would be to account for these commissions in specific storage variables rather than within the `Trader` entity, simplifying commission management and reducing potential errors.
##### Recommendation
We recommend creating specific storage variables to account for the administrator's commissions instead of relying on the `Trader` struct.

---
