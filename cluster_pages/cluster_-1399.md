# Cluster -1399

**Rank:** #179  
**Count:** 69  

## Label
Inconsistent balance validation and state updates lead to incorrect transaction thresholds, inflated totals, and flawed economic logic, enabling unauthorized claims, misaligned deposits, and potential denial-of-service.

## Cluster Information
- **Total Findings:** 69

## Examples

### Example 1

**Auto Label:** Inconsistent balance validation and state updates lead to incorrect transaction thresholds, inflated totals, and flawed economic logic, enabling unauthorized claims, misaligned deposits, and potential denial-of-service.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Medium

## Description

When a validator is deactivated via `ValidatorManager::deactivateValidator`, the validator’s balance is not updated. However, since a rebalance withdrawal request is created for this validator, their **actual** balance becomes 0.

```solidity

    function deactivateValidator(address validator) external whenNotPaused nonReentrant validatorExists(validator) {

        // ...

        Validator storage validatorData = _validators[index];

        require(validatorData.active, “Validator already inactive”);

        // Create withdrawal request before state changes

        if (validatorData.balance > 0) {

            _addRebalanceRequest(validator, validatorData.balance);

        }

        // Update state after withdrawal request

        validatorData.active = false;

        emit ValidatorDeactivated(validator);

    }

```

The problem occurs from the fact that even if his actual balance is 0, it will neverben updated since `OracleManager::updatePerformance()` passes the inactive validators without update. This could result to an inactive validator with his old non-zerobalance saved and this can be a problem because when this validator gets reactivate again through `ValidatorManager::reactivateValidator()` he will maintain in his old balance for a while.

The problem occurs from the fact that even though the validator’s actual balance is 0, it is never updated because `OracleManager::updatePerformance()` skips inactive validators. As a result, an inactive validator will retain its old non-zero balance in storage. This becomes problematic because when the validator is reactivated via `ValidatorManager::reactivateValidator()`, it will temporarily retain its outdated balance.

```solidity

    function reactivateValidator(address validator)

        // ...

    {

        // ...

        Validator storage validatorData = _validators[index];

        require(!validatorData.active, “Validator already active”);

        require(!_validatorsWithPendingRebalance.contains(validator), “Validator has pending rebalance”);

        // Reactivate the validator

        validatorData.active = true;

        emit ValidatorReactivated(validator);

    }

```

What is the impact of this? 

Firstly, `totalBalance` will be incorrectly inflated until the deactivated validator is reactivated and its balance is updated. This is because the validator receiving the redelegated amount will report an increased balance, and this increase will be included in `totalBalance` during `generatePerformance`.

Consider the following scenario:

- `valA:100, valB:200, totalBalance:300`

- `valA` is deactivated, and its balance is redelegated to `valB`.

- During the next update, `valB` will report a balance of `300`, causing totalBalance to increase to `400`. `valA`, which now has a balance of 0, will not report its balance because it is deactivated. 

This is incorrect because no new funds entered the system and only a balance transfer occurred. The system incorrectly tracks more funds than actually exist.

Secondly, in the above scenario, if `valA` is later reactivated, it will still have its old balance until `OracleManager::generatePerformance` updates it. If `valA`’s old balance is larger than the current `totalBalance`, a revert due to underflow will occur when `ValidatorManager::updateValidatorPerformance()` attempts to update `totalBalance`.

For example : 

- `valA` has a balance of 100 when deactivated.

- Some time later, `totalBalance` is reduced to 80.

- When `valA` is reactivated, `updateValidatorPerformance()` will attempt to compute :

```solidity

totalBalance = totalBalance - oldBalance + balance;

```

This translates to:

```solidity

totalBalance = 80 - 100 + 0

```

 This underflow causes a revert, resulting in a DoS on the update

```solidity

    function updateValidatorPerformance(

        address validator,

        uint256 balance,

        // ...

    ) external whenNotPaused onlyRole(ORACLE_ROLE) validatorExists(validator) validatorActive(validator) {

        // ...

        // Cache old balance for total balance update

        uint256 oldBalance = val.balance;

        // ...

        // Update total balance

@>        totalBalance = totalBalance - oldBalance + balance;

        // ...

    }

```

## Recommendations

Consider zeroing out the validator’s balance upon deactivation and subtracting it from `totalBalance` when the rebalance request is created. This will ensures that accounting inconsistencies and the potential underflows upon reactivation, will be avoided.

---
### Example 2

**Auto Label:** Inconsistent balance validation and state updates lead to incorrect transaction thresholds, inflated totals, and flawed economic logic, enabling unauthorized claims, misaligned deposits, and potential denial-of-service.  

**Original Text Preview:**

**Description:** In [`PerpetualBond::_validate`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/PerpetualBond.sol#L312-L314), there's a check to ensure that users have a non-zero balance before claiming yield:

```solidity
// Yield claim
require(balanceOf(_caller) > 0, "!bond balance"); // Caller must hold bonds to claim yield
require(accruedRewardAtCheckpoint[_caller] > 0, "!claimable yield"); // Must have claimable yield
```

However, this check can be bypassed by holding a trivial amount, such as 1 wei, of `PerpetualBond` tokens. A more meaningful check would ensure that the user's balance exceeds the `minimumTxnThreshold`, similar to how other parts of the contract enforce value-based thresholds.

Consider updating the balance check to compare against `minimumTxnThreshold` using the bond-converted value:

```diff
- require(balanceOf(_caller) > 0, "!bond balance");
+ require(_convertToBond(balanceOf(_caller)) > minimumTxnThreshold, "!bond balance");
```

Additionally, the second check on `accruedRewardAtCheckpoint[_caller]` is redundant, since [`PerpetualBond::requestYieldClaim`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/PerpetualBond.sol#L374-L378) already performs a value-based threshold check:

```solidity
// Convert yield amount to bond tokens for threshold comparison
uint256 yieldInBondTokens = _convertToBond(claimableYieldAmount);

// Check if the yield claim is worth executing
require(yieldInBondTokens >= minimumTxnThreshold, "!min txn threshold");
```

This makes the `accruedRewardAtCheckpoint` check in `_validate` unnecessary.

**YieldFi:** Fixed in commit [`f0bf88c`](https://github.com/YieldFiLabs/contracts/commit/f0bf88cb51a92a119cdde896c4b0118be1d1a031)

**Cyfrin:** Verified. Balance check removed as the user might still have yield even if they have no tokens (sold/transferred). Yield check in `_validate` is also removed as it's redundant.

\clearpage

---
### Example 3

**Auto Label:** Insufficient validation and improper state updates lead to unauthorized balance manipulation, incorrect resource accounting, and potential fund loss through race conditions and logic errors.  

**Original Text Preview:**

**Impact**

The system stores balances in storage and then updates them

Some tokens will charge a fee on transfer

Meaning that `_amount` stored in the `StoragePool` will be higher than the actual balance received

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/BorrowerOperations.sol#L841-L851

```solidity
  function _poolAddColl(
    address _borrower,
    IStoragePool _pool,
    address _collAddress,
    uint _amount,
    PoolType _poolType
  ) internal {
    _pool.addValue(_collAddress, true, _poolType, _amount);
    IERC20(_collAddress).transferFrom(_borrower, address(_pool), _amount); /// @audit FOT / SafeTransfer
  }

```

These types of tokens are pretty rare, but this is a very common finding that you should think about

**Mitigation**

Imo acknowledge this and make sure not to use these tokens

---
