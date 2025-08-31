# Cluster -1146

**Rank:** #217  
**Count:** 49  

## Label
Failure to properly handle and propagate errors leads to silent failures, misleading messages, or unhandled panics, compromising system reliability, observability, and security.

## Cluster Information
- **Total Findings:** 49

## Examples

### Example 1

**Auto Label:** Incorrect error handling leading to unreliable execution, silent failures, and exploitable panics due to failure to validate or properly propagate errors.  

**Original Text Preview:**

**Description:** Currently, the codebase imports the `erc7579-implementation` external repository at commit `42aa538397138e0858bae09d1bd1a1921aa24b8c `to use the `ExecutionHelper`. This import is used inside the `DelegationMetaSwapAdapter` to implement the `executeFromExecutor` method:
```solidity
    function executeFromExecutor(
        ModeCode _mode,
        bytes calldata _executionCalldata
    )
        external
        payable
        onlyDelegationManager
        returns (bytes[] memory returnData_)
    {
        (CallType callType_, ExecType execType_,,) = _mode.decode();

        // Check if calltype is batch or single
        if (callType_ == CALLTYPE_BATCH) {
            // Destructure executionCallData according to batched exec
            Execution[] calldata executions_ = _executionCalldata.decodeBatch();
            // check if execType is revert or try
            if (execType_ == EXECTYPE_DEFAULT) returnData_ = _execute(executions_);
            else if (execType_ == EXECTYPE_TRY) returnData_ = _tryExecute(executions_);
            else revert UnsupportedExecType(execType_);
        } else if (callType_ == CALLTYPE_SINGLE) {
            // Destructure executionCallData according to single exec
            (address target_, uint256 value_, bytes calldata callData_) = _executionCalldata.decodeSingle();
            returnData_ = new bytes[](1);
            bool success_;
            // check if execType is revert or try
            if (execType_ == EXECTYPE_DEFAULT) {
                returnData_[0] = _execute(target_, value_, callData_);
            } else if (execType_ == EXECTYPE_TRY) {
                (success_, returnData_[0]) = _tryExecute(target_, value_, callData_);
                if (!success_) emit TryExecuteUnsuccessful(0, returnData_[0]);
            } else {
                revert UnsupportedExecType(execType_);
            }
        } else {
            revert UnsupportedCallType(callType_);
        }
    }
```
When the `execType` is set to be `EXECTYPE_TRY`, the interal `_tryExecute` method is executed:
```solidity
    function _tryExecute(
        address target,
        uint256 value,
        bytes calldata callData
    )
        internal
        virtual
        returns (bool success, bytes memory result)
    {
        /// @solidity memory-safe-assembly
        assembly {
            result := mload(0x40)
            calldatacopy(result, callData.offset, callData.length)
@>          success := iszero(call(gas(), target, value, result, callData.length, codesize(), 0x00))
            mstore(result, returndatasize()) // Store the length.
            let o := add(result, 0x20)
            returndatacopy(o, 0x00, returndatasize()) // Copy the returndata.
            mstore(0x40, add(o, returndatasize())) // Allocate the memory.
        }
    }
```
This function executes a low level call and assigns the success the result of the call applying an is zero operator. This implementation is wrong because the result of a low level call already returns 1 on success and 0 on error. So applying an is zero operator will flip the result and be erroneus.
The result of this will be that when executing a transaction in try mode, successful executions will emit the `TryExecuteUnsuccessful` and failing executions will not. This can confuse the user if these events are used by the UI.

**Impact:** Incorrect event emission

**Recommended Mitigation:** Consider updating the commit version to the latest one, [it does not have this bug anymore](https://github.com/erc7579/erc7579-implementation/blob/main/src/core/ExecutionHelper.sol#L81)


**MetaMask:**
Resolved in commit [73e719](https://github.com/MetaMask/delegation-framework/commit/73e719dd25fe990d614f00203f089f4fd8b4761c).

**Cyfrin:** Resolved.

---
### Example 2

**Auto Label:** Failure to properly handle and propagate errors leads to silent failures, misleading messages, or unhandled panics, compromising system reliability, observability, and security.  

**Original Text Preview:**

## Security Report

## Severity
**Low Risk**

## Context
L1BlockInterop.sol#L29

## Description
The `isDeposit()` view function has a restriction that causes it to revert when called by any contract other than `CrossL2Inbox`. However, this restriction can be circumvented using the following approach:

1. Create a wrapper contract that calls `validateMessage` with an invalid message using a try/catch block.
2. The wrapper contract can then:
   - If the call reverts with the `NoExecutingDeposits`, bubble up the error.
   - If the call does not revert, revert with its own custom error message (thereby avoiding the supervisor marking the transaction as invalid due to the invalid message).

This technique allows the wrapper contract to effectively signal whether a transaction is a deposit or not. Subsequently, any contract can implement try/catch logic to process deposit and non-deposit transactions differently based on these revert messages.

## Recommendation
No solution seems viable within the normal context of the EVM.

## OP Labs
Acknowledged. This will be addressed in a redesign of `CrossL2Inbox` prior to release.

## Cantina Managed
Acknowledged.

---
### Example 3

**Auto Label:** Failure to properly handle and propagate errors leads to silent failures, misleading messages, or unhandled panics, compromising system reliability, observability, and security.  

**Original Text Preview:**

## Security Risk Assessment

## Severity: Low Risk

### Context
`FaultDisputeGame.sol#L1016`

### Description
The usual process for updating the Anchor State in the ASR (Anchor State Registry) is implemented as a call from an FDG (Fault Dispute Game) to the ASR. However, such call occurs within a try/catch statement that ignores reverts, resulting in the possibility of silent failures. Since that update is triggered by a permissionless function call to `closeGame()` in FDG, a malicious user might force an out of gas exception during the Anchor State update, while the overall call to `closeGame` succeeds.

The above scenario would result in the Anchor State not being updated by that FDG, possibly staying out of date for a length of time. The impact is limited given that `updateAnchorState` can be called in a permissionless way on the ASR, meaning any user aware of the problem can submit a transaction to perform the call and fix the issue.

### Recommendation
A process (ideally automated) should exist to track if the Anchor State hasn't been updated for some time, and call the `updateAnchorState` function so as to correctly update it.

### Optimism
Fixed in PR 14144. We chose not to change FaultDisputeGame but to instead introduce a fuzz test checking that the `closeGame()` function will either revert or correctly update the anchor state. Additionally, we've added a new semgrep rule that forces developers to explicitly mark try/catch blocks as "safe" from issues with EIP-150. We'll also be adding monitoring on the age of the anchor state in private repositories.

### Spearbit
Verified.

---
