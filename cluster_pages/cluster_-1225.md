# Cluster -1225

**Rank:** #303  
**Count:** 24  

## Label
Failure to properly deactivate or remove validators leads to persistent stake exposure, state mismatches, and incorrect economic calculations, creating liquidity risks and undermining protocol integrity.

## Cluster Information
- **Total Findings:** 24

## Examples

### Example 1

**Auto Label:** Failure to validate validator active status leads to unauthorized operations, fund leakage, or state inconsistencies, enabling malicious or erroneous withdrawals and blocking critical system functions.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

The `StakingManager` contract manages staking operations and validator delegation in the HYPE staking system. When users stake HYPE tokens through the `stake()` function, the funds are distributed via the internal `_distributeStake()` function, which handles both buffer management and delegation to validators.

Currently, `_distributeStake()` delegates funds to the validator address returned by `validatorManager.getDelegation()` without verifying if that validator is still active. This creates a risk where user funds could be delegated to validators that have been deactivated due to slashing or poor performance.

```solidity
        if (amount > 0) {
            address delegateTo = validatorManager.getDelegation(address(this));
            require(delegateTo != address(0), "No delegation set");

            // Send tokens to delegation
            l1Write.sendTokenDelegate(delegateTo, uint64(amount), false);

            emit Delegate(delegateTo, amount);
        }
```

This could lead to:
1. User funds being delegated to inactive/slashed validators
2. Reduced returns for stakers as inactive validators won't generate rewards

### Proof of Concept

1. A validator is active and set as the current delegation target
2. The validator gets slashed and deactivated via `OracleManager.generatePerformance()`
3. A user calls `stake()` with 10 HYPE
4. `_distributeStake()` delegates the funds to the deactivated validator
5. The user's stake is now delegated to an inactive validator that won't generate rewards

## Recommendations

Add a validation check in `_distributeStake()` to verify the validator's active status before delegation.

---
### Example 2

**Auto Label:** Failure to validate validator active status leads to unauthorized operations, fund leakage, or state inconsistencies, enabling malicious or erroneous withdrawals and blocking critical system functions.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** High

## Description
`StakingManager.deactivateValidator()` creates a withdrawal request but does not undelegate/withdraw funds from validator by calling `processValidatorWithdrawals()`:
```solidity
    function deactivateValidator(address validator) external whenNotPaused nonReentrant validatorExists(validator) {
        // Oracle manager will also call this, so limit the msg.sender to both MANAGER_ROLE and ORACLE_ROLE
        require(hasRole(MANAGER_ROLE, msg.sender) || hasRole(ORACLE_ROLE, msg.sender), "Not authorized");

        (bool exists, uint256 index) = _validatorIndexes.tryGet(validator);
        require(exists, "Validator does not exist");

        Validator storage validatorData = _validators[index];
        require(validatorData.active, "Validator already inactive");

        // Create withdrawal request before state changes
        if (validatorData.balance > 0) {
            _addRebalanceRequest(validator, validatorData.balance);
        }

        // Update state after withdrawal request
        validatorData.active = false;

        emit ValidatorDeactivated(validator);
    }
```
Because of this issue, funds will stuck in the validator’s account:
1. Manager call to `closeRebalanceRequests()` to remove the rebalance request will revert because there is not enough balance in staking manager.
2. Manager call to `rebalanceWithdrawal()` will revert because the validator is added to the pending list.
3. Sentinel call to `requestEmergencyWithdrawal()` will revert too for the same reason

## Recommendations
In `deactivateValidator()` function, withdraw funds from validator by calling: 
```IStakingManager(stakingManager).processValidatorWithdrawals();```

---
### Example 3

**Auto Label:** Failure to validate validator active status leads to unauthorized operations, fund leakage, or state inconsistencies, enabling malicious or erroneous withdrawals and blocking critical system functions.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Medium

## Description

`ValidatorManager._addRebalanceRequest()` checks validator's balance and records a withdrawal request:

```solidity
    function _addRebalanceRequest(address validator, uint256 withdrawalAmount) internal {
        require(!_validatorsWithPendingRebalance.contains(validator), "Validator has pending rebalance");
        require(withdrawalAmount > 0, "Invalid withdrawal amount");

        (bool exists, uint256 index) = _validatorIndexes.tryGet(validator);
        require(exists, "Validator does not exist");
        require(_validators[index].balance >= withdrawalAmount, "Insufficient balance");

        validatorRebalanceRequests[validator] = RebalanceRequest({validator: validator, amount: withdrawalAmount});
        _validatorsWithPendingRebalance.add(validator);

        emit RebalanceRequestAdded(validator, withdrawalAmount);
    }
```
However, a validator’s balance can change at any time due to rewards or slashing. This creates two potential problems when manager calls `closeRebalanceRequest()`:
1. Balance Decreases: If the validator’s balance decreases after the request is made, the delegation removal or withdrawal will fail.
2. Balance Increases: If the balance increases, some funds may remain in the validator’s account.

This is especially critical during emergency withdrawals or when deactivating a validator, as the manager needs to retrieve all funds.

## Recommendations

1. Ensure that the `closeRebalanceRequest()` function checks the validator’s balance to reflect the most up-to-date amount before processing the withdrawal request.
2. Add a mechanism to `closeRebalanceRequest()` so that manager can withdraw all available funds at the moment in validator balance

---
