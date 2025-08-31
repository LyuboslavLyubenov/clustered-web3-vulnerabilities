# Cluster -1019

**Rank:** #273  
**Count:** 30  

## Label
Improper session validation and state management lead to unauthorized access, session hijacking, and resource exhaustion through flawed access checks, incorrect timestamp logic, and unbounded iteration.

## Cluster Information
- **Total Findings:** 30

## Examples

### Example 1

**Auto Label:** Inconsistent session validation and access control across operations lead to unauthorized actions, session hijacking, and silent failures, enabling replay attacks and privilege escalation.  

**Original Text Preview:**

Within `SessionLib.validate`, it checks `transaction.to` to ensure it is not set to `msg.sender`, preventing callers from abusing session keys to administer the smart account. However, it allows `transaction.to` to be set to `SessionKeyValidator`, which could enable callers to create additional session keys with unrestricted calls if allowed.

```solidity
  function validate(
    SessionStorage storage state,
    Transaction calldata transaction,
    SessionSpec memory spec,
    uint64[] memory periodIds
  ) internal {
    // Here we additionally pass uint64[] periodId to check allowance limits
    // periodId is defined as block.timestamp / limit.period if limitType == Allowance, and 0 otherwise (which will be ignored).
    // periodIds[0] is for fee limit,
    // periodIds[1] is for value limit,
    // periodIds[2:] are for call constraints, if there are any.
    // It is required to pass them in (instead of computing via block.timestamp) since during validation
    // we can only assert the range of the timestamp, but not access its value.

    require(state.status[msg.sender] == Status.Active, "Session is not active");
    TimestampAsserterLocator.locate().assertTimestampInRange(0, spec.expiresAt);

    address target = address(uint160(transaction.to));

    if (transaction.paymasterInput.length >= 4) {
      bytes4 paymasterInputSelector = bytes4(transaction.paymasterInput[0:4]);
      require(
        paymasterInputSelector != IPaymasterFlow.approvalBased.selector,
        "Approval based paymaster flow not allowed"
      );
    }

    if (transaction.data.length >= 4) {
      // Disallow self-targeting transactions with session keys as these have the ability to administer
      // the smart account.
>>>   require(address(uint160(transaction.to)) != msg.sender, "Can not target self");

   // ...
  }
```

Consider reverting if `transaction.to` is set to the `SessionKeyValidator` address.

---
### Example 2

**Auto Label:** Improper session validation and state management lead to unauthorized access, session hijacking, and resource exhaustion through flawed access checks, incorrect timestamp logic, and unbounded iteration.  

**Original Text Preview:**

When `sessionStatus` and `sessionState` are called by the user to get the provided session key status, it will return `Active`, even if the session has already expired. This returned value could mislead users, implying that the session key can still be used. Consider adding an additional state (e.g., `Expired`), or returning `Closed` if the session has already expired.

---
### Example 3

**Auto Label:** Inconsistent session validation and access control across operations lead to unauthorized actions, session hijacking, and silent failures, enabling replay attacks and privilege escalation.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

When the module is removed, `disable` will be called after the module has been removed from the registered list.

```solidity
    function _removeModule(address module) internal {
        _modulesLinkedList().remove(module);

>>>     (bool success, ) = module.excessivelySafeCall(
            gasleft(),
            0,
            0,
            abi.encodeWithSelector(IInitable.disable.selector)
        );
        (success); // silence unused local variable warning

        emit RemoveModule(module);
    }
```

Inside `SessionKeyValidator`, it attempts to trigger `removeModuleValidator` and `removeHook`, which are restricted functions that can only be called via self-call or by registered modules.

```solidity
  function disable() external {
    if (_isInitialized(msg.sender)) {
      // Here we have to revoke all keys, so that if the module
      // is installed again later, there will be no active sessions from the past.
      // Problem: if there are too many keys, this will run out of gas.
      // Solution: before uninstalling, require that all keys are revoked manually.
      require(sessionCounter[msg.sender] == 0, "Revoke all keys first");
>>>   IValidatorManager(msg.sender).removeModuleValidator(address(this));
>>>   IHookManager(msg.sender).removeHook(address(this), true);
    }
  }
```

```solidity
   function removeModuleValidator(address validator) external onlySelfOrModule {
        _removeModuleValidator(validator);
    }
```

```solidity
    function removeHook(address hook, bool isValidation) external override onlySelfOrModule {
        _removeHook(hook, isValidation);
    }
```

But since `SessionKeyValidator` is no longer registered as a module, the calls will silently fail, as `removeModule` doesn't enforce the `disable` call to succeed. This will wrongly give the impression to users that `SessionKeyValidator` is no longer registered as a hook and module validator.

## Recommendations

Consider moving `disable` call before removing the module from the list.

---
