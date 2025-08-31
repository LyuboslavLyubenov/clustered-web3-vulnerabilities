# Cluster -1103

**Rank:** #45  
**Count:** 405  

## Label
Missing or flawed access control checks allow unauthorized token transfers or bypass of blacklisting, enabling front-running, fund withdrawal, or compliance violations through insufficient sender/recipient validation.

## Cluster Information
- **Total Findings:** 405

## Examples

### Example 1

**Auto Label:** Missing access controls and bound checks enable unauthorized operations and malicious state manipulation, leading to front-running, incorrect rewards, and exploitable vesting logic.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-06-superfluid-locker-system-judging/issues/207 

## Found by 
0x73696d616f, algiz, illoy\_sci

### Summary

The unlock flag is meant to signal that users can unlock SUP from their Locker. However, `provideLiquidity()` and `withdrawLiquidity()` are missing this flag, so users can unlock the SUP even with the unlock flag set to false.

### Root Cause

https://github.com/sherlock-audit/2025-06-superfluid-locker-system/blob/main/fluid/packages/contracts/src/FluidLocker.sol#L420
https://github.com/sherlock-audit/2025-06-superfluid-locker-system/blob/main/fluid/packages/contracts/src/FluidLocker.sol#L447
Provide liquidity and withdraw liquidity are missing the unlockAvailable modifier.

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

1. Locker has the unlock flag set to false.
2. User can provide liquidity, wait the tax free period and withdraw liquidity, while the unlock flag is still false.

### Impact

Users break key functionality.

### PoC

None

### Mitigation

Add the unlockAvailable modifier to these functions.

## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/superfluid-finance/fluid/pull/30

---
### Example 2

**Auto Label:** Missing access controls and bound checks enable unauthorized operations and malicious state manipulation, leading to front-running, incorrect rewards, and exploitable vesting logic.  

**Original Text Preview:**

## Severity

Low Risk

## Description

The DAO Manager can set extremely long vesting periods (e.g., 100 years) in the vesting schedule without any restrictions. This allows malicious DAO Managers to effectively lock users' tokens indefinitely, preventing them from ever claiming their tokens. As there is no check in the code, and the `daoManager` can be malicious, as this is not the protocol's trusted role.

## Location of Affected Code

File: [contracts/DaosVesting.sol](https://github.com/ED3N-Ventures/daoslive-sc/blob/9a1856db2060b609a17b24aa72ab35f2cdf09031/contracts/DaosVesting.sol)

## Recommendation

Implement a signature verification mechanism similar to the `contribute()` function, where the protocol verifier must sign off on the vesting schedule parameters. This ensures that vesting periods are reasonable and prevents malicious DAO Managers from setting excessive vesting lock periods.

## Team Response

Fixed.

---
### Example 3

**Auto Label:** Missing access controls and bound checks enable unauthorized operations and malicious state manipulation, leading to front-running, incorrect rewards, and exploitable vesting logic.  

**Original Text Preview:**

## Severity

Low Risk

## Description

The DAO Manager can manipulate vesting schedules by setting arbitrary start and end dates, potentially locking users' tokens for longer than necessary. For example, setting the first vesting period (June-July) with 50% unlock, then second period (October-November) with the remaining 50%, effectively locking tokens for 3 extra months, by leaving 3 months of Gap between the 2nd start date of the 2nd index.

## Location of Affected Code

File: [contracts/DaosVesting.sol](https://github.com/ED3N-Ventures/daoslive-sc/blob/9a1856db2060b609a17b24aa72ab35f2cdf09031/contracts/DaosVesting.sol)

## Recommendation

Implement two validations in the vesting schedule: the first vesting period must start at the current block timestamp, and each subsequent period's start date must match the previous period's end date. This ensures continuous vesting without unnecessary lock periods.

## Team Response

Fixed.

---
