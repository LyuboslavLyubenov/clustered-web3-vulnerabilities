# Cluster -1272

**Rank:** #410  
**Count:** 11  

## Label
Failure to validate input or contract compliance leads to silent failures, unauthorized operations, or exploitable state transitions, enabling theft, denial-of-service, or unintended token behavior.

## Cluster Information
- **Total Findings:** 11

## Examples

### Example 1

**Auto Label:** Failure to validate input lengths in precompiles leads to out-of-bounds access or incorrect state transitions, risking hard fork-level failures and blocked rollup or cross-chain operations.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-04-zetachain-cross-chain-judging/issues/328 

This issue has been acknowledged by the team but won't be fixed at this time.

## Found by 
berndartmueller, oade\_hacks, sherlockxyz

## Summary

The precompile caller can pass invalid (empty) calldata, which results in a panic in the precompile. This immediately exits the EVM and propagates up to the Cosmos SDK baseapp. The Cosmos SDK message (e.g., `MsgVoteOutbound`) fails, and the outbound cctx cannot be finalized. The cctx remains pending in the queue.

## Root Cause

In ZetaChain's precompiles, it is assumed that the provided `input` (calldata) by the caller is at least 4 bytes long (function selector).

This is observed in the following instances:

- [`precompiles/bank/bank.go#L95`](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/precompiles/bank/bank.go#L95)
- [`precompiles/bank/bank.go#L113`](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/precompiles/bank/bank.go#L113)
- [`precompiles/prototype/prototype.go#L96`](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/precompiles/prototype/prototype.go#L96)
- [`precompiles/prototype/prototype.go#L226`](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/precompiles/prototype/prototype.go#L226)
- [`precompiles/staking/staking.go#L122`](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/precompiles/staking/staking.go#L122)
- [`precompiles/staking/staking.go#L140`](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/precompiles/staking/staking.go#L140)

For example, the `RequiredGas()` function in the `bank` precompile,

```go
92: func (c *Contract) RequiredGas(input []byte) uint64 {
93: 	// get methodID (first 4 bytes)
94: 	var methodID [4]byte
95: 	copy(methodID[:], input[:4])
```

accesses the first 4 bytes of the `input` without checking if the length is at least 4 bytes. If the length is less than 4 bytes, this will panic with an `index out of range` error.

This panic is propagated up until the Cosmos SDK baseapp, where it is [caught](https://github.com/cosmos/cosmos-sdk/blob/7b9d2ff98d02bd5a7edd3b153dd577819cc1d777/baseapp/baseapp.go#L839-L847) to prevent the node from crashing.

However, the Cosmos SDK message has failed and cannot be executed.

## Internal Pre-conditions

None

## External Pre-conditions

None

## Attack Path

1. Attacker sends 500 ([MaxPendingCctxs = 500](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/x/crosschain/keeper/grpc_query_cctx.go#L19)) outbound CCTXs from ZetaChain to an external EVM chain (e.g., Ethereum) where they will error when calling the attacker's contract
2. Observers vote on the failed outbound CCTXs
3. The "final" CCTX vote attempts to finalize the outbound CCTX on ZetaChain by processing the revert on ZEVM
4. [As part of the revert process](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/x/crosschain/keeper/cctx_orchestrator_validate_outbound.go#L408-L418), the attacker contract's `onRevert()` function is [called](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/protocol-contracts/contracts/zevm/GatewayZEVM.sol#L463-L502)
5. In `onRevert()`, ZetaChain's precompile is called with invalid calldata (e.g., empty calldata)
6. The precompile panics, and the entire transaction fails
7. The outbound CCTX cannot be finalized and remains pending in the queue
8. This is repeated for all 500 outbound CCTXs, which remain pending in the queue
9. EVM zetaclients will be unable to process other outbound CCTXs [due to their lookback criteria](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/zetaclient/chains/evm/evm.go#L176-L183)

## Impact

- A Solidity contract that explicitly uses `try-catch` to ensure an external contract call must never cause the overall transaction to revert is unable to do so if the called contract calls the precompile with invalid calldata so that it panics.
- More severe, if the ZEVM call is initiated as part of the `MsgVoteOutbound` message in response to a failed outbound cctx (i.e., reverting the cctx on ZEVM), and it panics, the entire message fails. As a result, the outbound cctx cannot be finalized, it remains pending. An attacker can repeatedly cause multiple cctx's to remain in pending state so that the [pending cctx queue fills up until the limit is reached](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/x/crosschain/keeper/grpc_query_cctx.go#L193-L200) and the zetaclients are unable to process any new outbound cctx's.

## PoC

This simple PoC runs the `e2e/e2etests/test_precompiles_staking_through_contract.go` E2E test and demonstrates how the staking precompile is called with empty calldata so that it panics. The `try-catch` is just cosmetic as it is unable to catch the panic.

Update the `stakeWithStateUpdate` function in [`TestStaking.sol`](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/e2e/contracts/teststaking/TestStaking.sol) and add the new `causePanic` function:

```solidity
function stakeWithStateUpdate(
    address staker,
    string memory validator,
    uint256 amount
) external onlyOwner returns (bool) {
    counter = counter + 1;
    // transfer full balance
    (bool success, ) = payable(address(0xdead)).call{value: payable(this).balance }("");
    require(success, "transfer to dead address failed");

    success = staking.stake(staker, validator, amount);

    try this.causePanic() {} catch {
        // do nothing
    }

    counter = counter + 1;
    return success;
}

function causePanic() external {
    // call `staking` contract with low level call invalid calldata so that it panics
    address(staking).call(hex"");
}
```

Re-generate the ABI Go bindings:

```bash
go generate ./e2e/contracts/teststaking
```

Run the test:

```bash
zetae2e run precompile_contracts_staking_through_contract --config cmd/zetae2e/config/local.yml --verbose
```

The precompile will panic, and all state changes will be reverted. The Docker logs will show the panic:

```bash
panic recovered in runTx err="recovered: runtime error: slice bounds out of range [:4] with capacity 0
```

## Mitigation

Consider checking that the `input` length is at least 4 bytes.

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

**Auto Label:** Failure to validate input or contract compliance leads to silent failures, unauthorized operations, or exploitable state transitions, enabling theft, denial-of-service, or unintended token behavior.  

**Original Text Preview:**

##### Description

The `BellumFactory` contract approves an unlimited amount (i.e.: `type(uint256).max`) of newly created tokens to external DEX routers (`TJ_ROUTER` or `PHARAOH_ROUTER`). While this is done for convenience, it creates a security risk:

  

If either router contract is compromised through:

* Implementation vulnerabilities
* Upgradeable proxy attacks
* Admin key compromise
* Future bugs

  

The attacker could:

* Drain all tokens held by the BellumFactory
* Manipulate token launches
* Extract value from the protocol
* Impact multiple tokens simultaneously

##### BVSS

[AO:A/AC:L/AX:M/R:N/S:U/C:N/A:N/I:H/D:M/Y:L (6.3)](/bvss?q=AO:A/AC:L/AX:M/R:N/S:U/C:N/A:N/I:H/D:M/Y:L)

##### Recommendation

Replace infinite approvals with exact amount approvals during token launch.

##### Remediation

**SOLVED**: The suggested mitigation was implemented by the **Bellum Exchange team**.

##### Remediation Hash

24b6f9b4508a1d5e5ae26923a5bdac602841c939

---
