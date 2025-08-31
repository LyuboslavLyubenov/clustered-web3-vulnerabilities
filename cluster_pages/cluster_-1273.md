# Cluster -1273

**Rank:** #431  
**Count:** 9  

## Label
Failure to validate modules against a registry at runtime due to out-of-order execution, enabling unauthorized or revoked modules to operate with unchecked access.

## Cluster Information
- **Total Findings:** 9

## Examples

### Example 1

**Auto Label:** Signature validation flaws enabling unauthorized module installation and replay, bypassing access controls through omitted or invalid module type checks.  

**Original Text Preview:**

## Security Analysis Report

## Severity
**Low Risk**

## Context
(No context files were provided by the reviewer)

## Description
A validator that does not meet the minimum threshold of attestors in the ERC-7484 registry can be used to pass validation through "enable mode" during `validateUserOp`. Enable mode is intended as a feature whereby a new validator module can be installed and immediately used for validation of a user operation when `validateUserOp` is called. Initially, the nonce is checked to see if enable mode has been requested, in which case the `_enableMode` internal function is called.

```solidity
function validateUserOp(
    PackedUserOperation calldata op,
    bytes32 userOpHash,
    uint256 missingAccountFunds
) external virtual payPrefund(missingAccountFunds) onlyEntryPoint returns (uint256 validationData) {
    address validator = op.nonce.getValidator();
    if (!op.nonce.isModuleEnableMode()) {
        // Check if validator is not enabled. If not, return VALIDATION_FAILED.
        if (!_isValidatorInstalled(validator)) return VALIDATION_FAILED;
        validationData = IValidator(validator).validateUserOp(op, userOpHash);
    } else {
        PackedUserOperation memory userOp = op;
        userOp.signature = _enableMode(validator, op.signature);
        validationData = IValidator(validator).validateUserOp(userOp, userOpHash);
    }
}
```

However, the `enableMode` function logic does not check if the type of module is a validator. Therefore, as long as the signature is valid, any type of module can be installed through enable mode, not just a validator type.

```solidity
function _enableMode(address module, bytes calldata packedData) internal returns (bytes calldata userOpSignature) {
    uint256 moduleType;
    bytes calldata moduleInitData;
    bytes calldata enableModeSignature;
    (moduleType, moduleInitData, enableModeSignature, userOpSignature) = packedData.parseEnableModeData();
    _checkEnableModeSignature(
        _getEnableModeDataHash(module, moduleInitData),
        enableModeSignature
    );
    _installModule(moduleType, module, moduleInitData);
}
```

Upon installation of the new module, the control flow continues with the last line in `validateUserOp()` which calls the new module.

```solidity
userOp.signature = _enableMode(validator, op.signature);
validationData = IValidator(validator).validateUserOp(userOp, userOpHash);
```

Multi-type modules can be both validators and other types of modules as well. Each type must be attested to on the 7484 registry. A multi-type module may have many attestors in one module-type and few in another. 

For example, a multi-type module may meet the attestor threshold as a `MODULE_TYPE_EXECUTOR` module but not as a `MODULE_TYPE_VALIDATOR`. When this type of module is passed in enable mode with the type `EXECUTOR` selected, it will be successfully installed (as an executor) and then immediately used as a validator. Normally, upon installation of a new validator, there is a check made on the registry for the module address and module type `VALIDATOR`. But in this case, since it is being installed as an executor, it will not check the validator type.

## Recommendation
Consider adding the same validator validation used above in the non-enable-mode case:

```solidity
} else {
    PackedUserOperation memory userOp = op;
    userOp.signature = _enableMode(validator, op.signature);
    + if (!_isValidatorInstalled(validator)) return VALIDATION_FAILED;
    validationData = IValidator(validator).validateUserOp(userOp, userOpHash);
}
```

Alternatively, consider including the module type in the `enableMode` signature hash along with a check that the module is a validator type.

## Acknowledgments
- **Biconomy:** Fixed in PR 129 and PR 126.
- **Spearbit:** Fixed.

---
### Example 2

**Auto Label:** Failure to validate modules against a registry at runtime due to out-of-order execution, enabling unauthorized or revoked modules to operate with unchecked access.  

**Original Text Preview:**

## Security Report

## Severity
**Low Risk**

## Context
`RegistryBootstrap.sol#L46`

## Description
The `Bootstrap.init*` functions install modules before setting the registry. This will skip the registry check on these modules. Modules that are not validated could be installed during initialization.

## Recommendation
Set the registry before installing modules.

## Biconomy
Fixed in PR 115.

## Spearbit
Fixed.

---
### Example 3

**Auto Label:** Signature validation flaws enabling unauthorized module installation and replay, bypassing access controls through omitted or invalid module type checks.  

**Original Text Preview:**

## Severity: Medium Risk

## Context
(No context files were provided by the reviewer)

## Description
According to the documentation, "Any module type can be installed via Module Enable Mode". This mode is meant for installing a module from within `validateUserOp()` before it gets used in the following execution defined by the `userOp`. This is helpful in cases where the account owner does not want to issue an explicit `installModule` in the `userOp` callData and for use cases where a module installation during the validation might be necessary for the execution to work, as highlighted in the documentation.

### Module Installation in Enable Mode
This is how modules are installed in enable mode:

```solidity
function validateUserOp(
    PackedUserOperation calldata op,
    bytes32 userOpHash,
    uint256 missingAccountFunds
) external virtual payPrefund(missingAccountFunds) onlyEntryPoint returns (uint256 validationData) {
    address validator = op.nonce.getValidator();
    if (!op.nonce.isModuleEnableMode()) {
        // Check if validator is not enabled. If not, return VALIDATION_FAILED.
        if (!_isValidatorInstalled(validator)) return VALIDATION_FAILED;
        validationData = IValidator(validator).validateUserOp(op, userOpHash);
    } else {
        PackedUserOperation memory userOp = op;
        userOp.signature = _enableMode(validator, op.signature);
        validationData = IValidator(validator).validateUserOp(userOp, userOpHash);
    }
}
```

Contrary to the documentation, this logic will only allow validator modules and multi-type modules with a validator component to be installed in enable mode. This is because the module address that is to be installed is extracted from `nonce`, and then the same module address is used for the validation of `userOp`. This is incorrect because other module types (hooks, executors, and fallback handlers) cannot be expected to validate the `userOp`. Any standard implementation of these modules would not have a `validateUserOp()` function. 

Biconomy intends to allow any module type to be installed in the "Module Enable Mode," but the current logic results in broken functionality.

## Recommendation
Currently, the module-to-be-installed will always be used as a validator. Enable mode makes most sense for validators as other module types can be installed via batching as the documentation acknowledges:

>If the module we want to install before the usage is executor, this can be simply solved by batching `installModule(executor)` call before the call that goes through this executor. However, that would not work for validators, as the validation phase in EIP-4337 goes before execution, thus the new validator should be enabled before it is used for the validation. Enable Mode allows to enable the module in the beginning of the validation phase. 

To achieve this, the user signs the data object that describes which module is going to be installed and how it should be configured during the installation. Any module type can be installed via Module Enable Mode; however, we believe it makes most sense for validators and hooks.

Consider restricting enable mode to validator module type only; otherwise, the current approach of always using the module as a validator needs to be changed significantly.

## Biconomy
Fixed in PR 112.

## Spearbit
Fixed; there are now 3 modules:
- The outer validator that validates the user op, coming from the `userOp.nonce`.
- The `enableModeValidator` and the module to be installed, both coming from `userOp.signature`.

If we want to install `moduleType = validator`, we can set `module = validator` and use the new module to validate the `userOp` (that's the main use case for enable mode). If we want to install a non-validator module, these three validators can all be different.

---
