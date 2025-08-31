# Cluster -1180

**Rank:** #240  
**Count:** 40  

## Label
Inconsistent state handling during validator updates leads to incorrect performance calculations, stale balances, and denial-of-service conditions by failing to synchronize or properly account for balance and status changes.

## Cluster Information
- **Total Findings:** 40

## Examples

### Example 1

**Auto Label:** Inconsistent state handling during validator updates leads to incorrect performance calculations, stale balances, and denial-of-service conditions by failing to synchronize or properly account for balance and status changes.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

In OracleManager, the operator role will generate validators' performance result periodically. When we get the performance result, we will check the validator's behavior. If this validator's behavior is below expectation, we will deactivate this validator.

The problem is that we will trigger to deactivate the validator directly, missing updating the latest slash/reward/balance amount.

There are some possible impacts:
1. If there are more rewards than slash in the latest period, we will withdraw less HYPE than we expect.
2. If there are more slash than rewards in the latest period, we will withdraw more HYPE than we delegate in the validator. This will cause the un-delegate failure.
3. In StakingManager, we will make use of `slash` and `reward` amount to calculate the exchange ratio. The calculation will be inaccurate.

```solidity
    function generatePerformance() external whenNotPaused onlyRole(OPERATOR_ROLE) returns (bool) {
        if (
            !_checkValidatorBehavior(
                validator, previousSlashing, previousRewards, avgSlashAmount, avgRewardAmount, avgBalance
            )
        ) {
            validatorManager.deactivateValidator(validator);
            continue;
        }

        if (avgSlashAmount > previousSlashing) {
            uint256 newSlashAmount = avgSlashAmount - previousSlashing;
            validatorManager.reportSlashingEvent(validator, newSlashAmount);
        }

        if (avgRewardAmount > previousRewards) {
            uint256 newRewardAmount = avgRewardAmount - previousRewards;
            validatorManager.reportRewardEvent(validator, newRewardAmount);
        }

        validatorManager.updateValidatorPerformance(
            validator, avgBalance, avgUptimeScore, avgSpeedScore, avgIntegrityScore, avgSelfStakeScore
        );
    }
```


## Recommendations

Update the balance/reward/slash to the latest amount, then deactivate this validator.

---
### Example 2

**Auto Label:** Inconsistent state handling during validator updates leads to incorrect performance calculations, stale balances, and denial-of-service conditions by failing to synchronize or properly account for balance and status changes.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Low

## Description

`OracleManager::generatePerformance` is intended to update the stats of every validator and the `totalBalance` in `ValidatorManager`. This function is called once an hour (or at whatever interval is set by `MIN_UPDATE_INTERVAL`). It is crucial that this call does not revert, as a revert would cause the `ValidatorManager` to remain in a stale state. Additionally, `OracleManager::generatePerformance` performs an abnormal behaviour check, and if necessary, deactivates a validator.
```solidity
        // Check for anomalies
        if (
            !_checkValidatorBehavior(
                validator, previousSlashing, previousRewards, avgSlashAmount, avgRewardAmount, avgBalance
            )
        ) {
            // Deactivate validator if behavior is unexpected
@>            validatorManager.deactivateValidator(validator);
            continue;
        }
```
However, this deactivation call can easily revert (and thus cause the entire update to fail) if the given validator has an active rebalance request.
```solidity
    function deactivateValidator(address validator) external whenNotPaused nonReentrant validatorExists(validator) {
        // ...

        // Create withdrawal request before state changes
        if (validatorData.balance > 0) {
@>            _addRebalanceRequest(validator, validatorData.balance);
        }

        // ...
    }

    function _addRebalanceRequest(address validator, uint256 withdrawalAmount) internal {
@>        require(!_validatorsWithPendingRebalance.contains(validator), "Validator has pending rebalance");
        // ...
    }
```
The result of this issue is a temporary DoS that prevents the performance update for validators from proceeding until the rebalance withdrawal request for the affected validator is resolved. In a large-scale system with many validators, such scenarios are likely to occur.

## Recommendations

One way to mitigate this issue is to zero out the validator’s balance (and subtract it from `totalBalance`) when a rebalance request is created. This way, upon deactivation, the validator would not attempt to open a new rebalance request.

---
### Example 3

**Auto Label:** Inconsistent state handling during validator updates leads to incorrect performance calculations, stale balances, and denial-of-service conditions by failing to synchronize or properly account for balance and status changes.  

**Original Text Preview:**

## Medium Risk Report

## Severity
Medium Risk

## Context
OracleManager.sol#L165

## Description
The `_checkValidatorBehavior()` function implements checks to ensure that the changes in the slashed amount and rewards amount are reasonable. To achieve this, tolerance levels are defined for both variables. 

In practice, these tolerance levels are represented as percentages of the `avgBalance` parameter, which is the current balance of the validator (after the change). 

To better understand the issue, let's consider a simple example:

- **SlashingTolerance** = 10%.
- Current balance of validator A is 100 (already stored inside the ValidatorManager contract).
- At this point, there is a slashing of 10.
- For simplicity, assuming all oracles submit the same values, in the call to `generatePerformance()`, we will have:
  - `avgBalance` = 90.
  - `avgSlashAmount` = 10.

The issue here is that the call to `_checkValidatorBehavior()` will return false (instead of true) since the computation will be:

```
slashingBps = (10 / 90) % > SlashingTolerance (10%).
```

## Recommendation
Consider fetching the balance before the change by calling `ValidatorManager.validatorPerformance()` and using it for `_checkValidatorBehavior()` instead.

## Acknowledgements
- Kinetiq: Acknowledged.
- Cantina Managed: Acknowledged.

---
