# Cluster -1150

**Rank:** #245  
**Count:** 38  

## Label
Failure to validate or check contract state before execution leads to incorrect state updates, unauthorized operations, or silent failures, risking asset loss, inconsistent accounting, or exploitation during invalid or transitional states.

## Cluster Information
- **Total Findings:** 38

## Examples

### Example 1

**Auto Label:** Inadequate validation and poor runtime checks lead to incorrect behavior, gas inefficiency, and potential state corruption under invalid or edge inputs.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-04-zetachain-cross-chain-judging/issues/370 

## Found by 
0x73696d616f, g

### Summary

There are cases when a transaction for the Gateway will skip the compute phase. When this happens, [`exitCodeFromTx()`](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/pkg/contracts/ton/gateway_parse.go#L291-L293) will panic because `TrPhaseComputeVm` will be nil. 

### Root Cause

[`exitCodeFromTx()`](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/pkg/contracts/ton/gateway_parse.go#L291-L293) assumes that `TrPhaseComputeVm` is always present. However, that is false because `ComputePh` is a Sum type with two variants.

```golang
type TrComputePhase struct {
	SumType
	TrPhaseComputeSkipped struct {
		Reason ComputeSkipReason
	} `tlbSumType:"tr_phase_compute_skipped$0"`
	TrPhaseComputeVm struct {
                     // ... snip ...
		} `tlb:"^"`
	} `tlbSumType:"tr_phase_compute_vm$1"`
}
```

A transaction can skip the compute phase when the message sent to the Gateway contract provides less gas than is required for minimal execution. The minimum amount of gas needed is implementation-dependent, but roughly in the ~10⁵ nanoton (0.1 TON) range for basic contracts.

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

1. A malicious user sends a message with very low value, e.g., 0.001 TON. Set the `forward_fee` or `fee_remaining` so that only a tiny gas budget is passed. The message will have a valid message body for a deposit operation.
2. TON VM will then skip execution and record `compute.skipped = true` for the message sent to the Gateway contract.
3. The TON Observer will [get](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/zetaclient/chains/ton/observer/inbound.go#L44-L65) every transaction and parse it. 
4. When parsing the Inbound Deposit that skipped the compute phase, it will panic when calling [`exitCodeFromTx()`](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/pkg/contracts/ton/gateway_parse.go#L71).

### Impact

All TON Observers' `ObserveInbound` and `ProcessInboundTrackers` go-routines will crash and no longer process any inbound deposits to the TON Gateway.

### PoC

_No response_

### Mitigation

Consider adding a check to ensure the compute phase is present before accessing the `TrPhaseComputeVm` field.

---
### Example 2

**Auto Label:** Failure to validate or check contract state before execution leads to incorrect state updates, unauthorized operations, or silent failures, risking asset loss, inconsistent accounting, or exploitation during invalid or transitional states.  

**Original Text Preview:**

##### Description

The logic in the `stake` function includes two inefficiencies:

  

1. **Redundant Balance Comparison**: The function calculates `stakingAmount` by comparing the contract’s token balance before and after the transfer. This approach is unnecessary and gas-inefficient. Instead, the `_amount` parameter can be used directly, or it can be replaced with more accurate validations, such as checking the sender's balance and allowance.
2. **Unnecessary Conditional Update**: The user’s stake (`userStake.amount`) is updated with conditional logic based on whether `userStake.amount > 0`. This separation is redundant, as a direct addition would achieve the same result with less complexity.

  

Code Location
-------------

Code of `stake` function from **Staking.sol** contract.

```
Stake storage userStake = stakeInfo[_beneficiary];

uint256 balanceBefore = IERC20(stakingToken).balanceOf(address(this));
IERC20(stakingToken).safeTransferFrom(
	msg.sender,
	address(this),
	_amount
);
uint256 balanceAfter = IERC20(stakingToken).balanceOf(address(this));

uint256 stakingAmount = balanceAfter - balanceBefore;

// Update user's stake
if (userStake.amount > 0) {
	userStake.amount += stakingAmount;
} else {
	userStake.amount = stakingAmount;
}

totalStaked += stakingAmount;
```

##### BVSS

[AO:A/AC:L/AX:L/R:N/S:U/C:N/A:N/I:N/D:N/Y:N (0.0)](/bvss?q=AO:A/AC:L/AX:L/R:N/S:U/C:N/A:N/I:N/D:N/Y:N)

##### Recommendation

It is recommended to:

  

1. Replace the balance comparison logic with either directly using the `_amount` parameter or validating the sender’s balance and allowance beforehand to ensure they have sufficient tokens and approval.
2. Simplify the logic for updating `userStake.amount` by directly adding the staking amount to the existing value, regardless of whether it is initially zero.

##### Remediation

**ACKNOWLEDGED:** The **Plume team** has acknowledged this finding.

---
### Example 3

**Auto Label:** Improper validation of contract existence or transaction outcomes leads to permanent fund loss or unauthorized access due to insufficient state checks and lack of error handling.  

**Original Text Preview:**

## TokenPayAndSettleIncoming Functionality

## Context
File: `channel_manager.rs`  
Lines: 719-724  

## Description
The `TokenPayAndSettleIncoming` functionality processes incoming transfers from EVM. It implements multiple checks on the expected fields. However, one of these checks is insufficient. The EVM channel checks that the hashlock value is not `[0u8; 32]` before processing the transfer. This check must be hardened to ensure that the hashlock value corresponds to the expected hashlock. The current implementation can lead the plugin to process invalid transfers.

## Recommendation
Ensure that the hashlock in the EVM smart contract corresponds to the hashlock passed in the request parameters.

## Resolution
- **SatsBridge**: Fixed in commit `f3b213b4`.
- **Cantina Managed**: Fixed. The hashlock is now correctly checked.

---
