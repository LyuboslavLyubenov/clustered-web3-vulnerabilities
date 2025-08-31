# Cluster -1183

**Rank:** #198  
**Count:** 56  

## Label
Failure to validate or check contract state before execution leads to incorrect state updates, unauthorized operations, or silent failures, risking asset loss, inconsistent accounting, or exploitation during invalid or transitional states.

## Cluster Information
- **Total Findings:** 56

## Examples

### Example 1

**Auto Label:** Failure to validate external contract states (like pausing) leads to cascading operation failures, permanent user lockouts, or denial-of-service, undermining system availability and user access.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

In `LiqFacet`, when users add liquidity, they deposit a single token into a specific vault:

```solidity
    function addLiq(
        address recipient,
        uint16 _closureId,
        address token,
        uint128 amount
    ) external nonReentrant returns (uint256 shares) {
```

However, when removing liquidity, the function attempts to withdraw from all vaults where the token is in the closure:

```solidity
        for (uint8 i = 0; i < n; ++i) {
            VertexId v = newVertexId(i);
            VaultPointer memory vPtr = VaultLib.get(v);
            uint128 bal = vPtr.balance(cid, false);
            if (bal == 0) continue;
            // If there are tokens, we withdraw.
            uint256 withdraw = FullMath.mulX256(percentX256, bal, false);
            vPtr.withdraw(cid, withdraw);
            vPtr.commit();
            address token = tokenReg.tokens[i];
            TransferHelper.safeTransfer(token, recipient, withdraw);
        }
```

The issue arises when one of these vaults is paused. Since the function does not account for paused vaults, the entire withdrawal process is blocked, preventing users from removing liquidity.

## Recommendations

Implement a mechanism to skip paused vaults and allow withdrawals from the remaining active vaults to ensure a smooth liquidity removal.

---
### Example 2

**Auto Label:** Failure to validate input or contract compliance leads to silent failures, unauthorized operations, or exploitable state transitions, enabling theft, denial-of-service, or unintended token behavior.  

**Original Text Preview:**

...

Export to GitHub ...

Set external GitHub Repo ...

Export to Clipboard (json)

Export to Clipboard (text)

#### Resolution

Acknowledged by the client with the following note:

> We added a warning in the documentation in this hash: [bfca5f9163eb4c5b1a1ff309a5fc9d6a5815f538](https://github.com/MetaMask/delegation-framework/commit/bfca5f9163eb4c5b1a1ff309a5fc9d6a5815f538)

#### Description

In the `beforeHook` function of the `MultiTokenPeriodEnforcer` contract, there is a validation that the mode of execution is `CALLTYPE_SINGLE` and `EXECTYPE_DEFAULT`, so the ERC-20 token transfers will always be executed via `_execute` function, which performs a low-level `call`. However, the function lacks critical safety checks typically found in secure token transfers. There is no validation that the token transfer succeeded by returning `true`, as required by the ERC-20 standard. This omission could result in silent failures when interacting with non-standard token contracts, allowing tokens that don’t revert on failures and just return `false` to be incorrectly treated as successfully transferred.

#### Examples

**src/enforcers/MultiTokenPeriodEnforcer.sol:L131-L144**

```
function beforeHook(
    bytes calldata _terms,
    bytes calldata,
    ModeCode _mode,
    bytes calldata _executionCallData,
    bytes32 _delegationHash,
    address,
    address _redeemer
)
    public
    override
    onlySingleCallTypeMode(_mode)
    onlyDefaultExecutionMode(_mode)
{

```

**src/DeleGatorCore.sol:L182-L216**

```
 * @notice Executes an Execution from this contract
 * @dev Related: @erc7579/MSAAdvanced.sol
 * @dev This method is intended to be called through a UserOp which ensures the invoker has sufficient permissions
 * @param _mode The ModeCode for the execution
 * @param _executionCalldata The calldata for the execution
 */
function execute(ModeCode _mode, bytes calldata _executionCalldata) external payable onlyEntryPoint {
    (CallType callType_, ExecType execType_,,) = _mode.decode();

    // Check if calltype is batch or single
    if (callType_ == CALLTYPE_BATCH) {
        // destructure executionCallData according to batched exec
        Execution[] calldata executions_ = _executionCalldata.decodeBatch();
        // Check if execType is revert or try
        if (execType_ == EXECTYPE_DEFAULT) _execute(executions_);
        else if (execType_ == EXECTYPE_TRY) _tryExecute(executions_);
        else revert UnsupportedExecType(execType_);
    } else if (callType_ == CALLTYPE_SINGLE) {
        // Destructure executionCallData according to single exec
        (address target_, uint256 value_, bytes calldata callData_) = _executionCalldata.decodeSingle();
        // Check if execType is revert or try
        if (execType_ == EXECTYPE_DEFAULT) {
            _execute(target_, value_, callData_);
        } else if (execType_ == EXECTYPE_TRY) {
            bytes[] memory returnData_ = new bytes[](1);
            bool success_;
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

```
// from erc7579-implementation/src/core/ExecutionHelper.sol

    function _execute(
        address target,
        uint256 value,
        bytes calldata callData
    )
        internal
        virtual
        returns (bytes memory result)
    {
        /// @solidity memory-safe-assembly
        assembly {
            result := mload(0x40)
            calldatacopy(result, callData.offset, callData.length)
            if iszero(call(gas(), target, value, result, callData.length, codesize(), 0x00)) {
                // Bubble up the revert if the call reverts.
                returndatacopy(result, 0x00, returndatasize())
                revert(result, returndatasize())
            }
            mstore(result, returndatasize()) // Store the length.
            let o := add(result, 0x20)
            returndatacopy(o, 0x00, returndatasize()) // Copy the returndata.
            mstore(0x40, add(o, returndatasize())) // Allocate the memory.
        }
    }

```

#### Recommendation

We recommend creating a separate execution logic for ERC-20 token transfers, which will validate all the edge case scenarios:

* Even though with the current code a call to EOA will revert, we still recommend verifying that the target address is a contract before executing the low-level `call`, e.g., using `Address.isContract`.
* In cases when the token returns a variable, checking that the token transfer returns `true` or does not return `false`, in line with the ERC-20 specification, as it is done in the SafeERC20 library.

---
### Example 3

**Auto Label:** Failure to validate critical state conditions leads to permanent fund lock or system inaccessibility, enabling attackers or owners to halt operations without recovery mechanisms.  

**Original Text Preview:**

## Low Risk Vulnerability Report

## Severity
Low Risk

## Context
- **BuidlUSDCSource.sol**: Lines 202-210
- **PSMSource.sol**: Lines 186-201

## Description
Due to how the `OndoTokenRouter` contract is set up, it expects the result of the `availableToWithdraw` call to be available for withdrawal from all the token sources. If the `availableToWithdraw` function returns a non-zero value but this amount is not actually obtainable by calling the `tokenSource.withdrawToken` function, it can lead to reverts or broken accounting.

```solidity
uint256 amountAvailable = tokenSource.availableToWithdraw(outputToken);
uint256 withdrawAmount = amountAvailable > requestedWithdrawAmount
    ? requestedWithdrawAmount
    : amountAvailable;

// INVARIANT - `TokenSource.withdrawToken` will always withdraw the amount requested
// or revert.
tokenSource.withdrawToken(outputToken, withdrawAmount);
```

Some of the token source contracts are pausable. In case they are paused, the amount reported should be 0, so that the router does not try to extract tokens from it.

The `BuidlUSDCSource` contract reports the available liquidity in the BUIDL settlement contract but doesn't check if the contracts are paused, and thus, if this amount is actually obtainable. If the BUIDL redemption is paused, the `withdrawToken` calls to this token source contract can revert.

Similarly, the `PSMSource` contract uses the USDS PSM module which implements a tin parameter that assigns the paused state. This can be verified at the following address:
- `0xA188EEC8F81263234dA3622A406892F3D630f98c`

Calls to this contract can also revert when it is paused.

## Proof of Concept
Assume the `BuidlUSDCSource` has 100 USDC tokens, 900 BUIDL tokens, and the BUIDL settlement contract has enough liquidity but is paused. The `availableToWithdraw` function will report 1000 USDC. However, the contract can only dispense up to 100 USDC. Any more, and the redeem function will be called on the BUIDL redeemer, which will cause the transaction to revert since it is paused.

## Recommendation
Check the paused state of underlying contracts. If paused, only the current USDC balance in the token source contract is available, and any redemptions/swaps are offline.

## Additional Information
- **Ondo Finance**: Fixed in commit `a876a50b`.
- **Spearbit**: Fix verified.

---
