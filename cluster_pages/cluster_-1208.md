# Cluster -1208

**Rank:** #242  
**Count:** 39  

## Label
Inadequate validation and improper state management in linked list operations lead to dangling references, loops, and unauthorized account inclusion, causing incorrect state calculations and potential infinite loops or unauthorized access.

## Cluster Information
- **Total Findings:** 39

## Examples

### Example 1

**Auto Label:** Failure to validate edge cases in linked list or state management leads to runtime errors, memory leaks, and inconsistent state due to improper null checks or pointer updates.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-04-zetachain-cross-chain-judging/issues/326 

## Found by 
berndartmueller

## Summary

By providing an additional account to the Solana gateway `call` function, the Solana zetaclient will error, preventing processing other inbound Solana transactions.

## Root Cause

The Solana gateway `call` function expects one account, the signer, to be passed in the instruction.

[`programs/gateway/src/contexts.rs#L109-L114`](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/protocol-contracts-solana/programs/gateway/src/contexts.rs#L109-L114)

```rust
109: #[derive(Accounts)]
110: pub struct Call<'info> {
111:     /// The account of the signer making the call.
112:     #[account(mut)]
113:     pub signer: Signer<'info>,
114: }
```

Additional accounts can be passed by the user, they will just be ignored by the program.

However, the Solana zetaclient expects the instruction to have exactly one signer account. If there is more than one signer account, it errors out with the message `want only 1 signer account, got %d`.

[`pkg/contracts/solana/inbound.go#L264-L266`](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/pkg/contracts/solana/inbound.go#L264-L266)

```go
238: func ParseInboundAsCall(
239: 	tx *solana.Transaction,
240: 	instructionIndex int,
241: 	slot uint64,
242: ) (*Inbound, error) {
...   // [...]
257:
258: 	// get the sender address (skip if unable to parse signer address)
259: 	instructionAccounts, err := instruction.ResolveInstructionAccounts(&tx.Message)
260: 	if err != nil {
261: 		return nil, err
262: 	}
263:
264: 	if len(instructionAccounts) != 1 {
265: 		return nil, fmt.Errorf("want only 1 signer account, got %d", len(instructionAccounts))
266: 	}
```

This error is propagated up to the `FilterInboundEventsAndVote` function, which is called in the `ObserveInbound` function.

[`zetaclient/chains/solana/observer/inbound.go#L88-L92`](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/zetaclient/chains/solana/observer/inbound.go#L88-L92)

```go
88: // filter inbound events and vote
89: if err = ob.FilterInboundEventsAndVote(ctx, txResult); err != nil {
90: 	// we have to re-scan this signature on next ticker
91: 	return errors.Wrapf(err, "error FilterInboundEventAndVote for sig %s", sigString)
92: }
```

Due to this error, the same transaction will be re-scanned on the next ticker, preventing other inbound Solana transactions from being processed indefinitely.

## Internal Pre-conditions

None

## External Pre-conditions

None

## Attack Path

1. The attacker sends a Solana transaction with the `call` instruction to the Solana gateway program with an additional account.
2. The Solana transaction is executed, and the `call` instruction is processed successfully.
3. The Solana zetaclient processes the transaction and calls the `ParseInboundAsCall` function. The check `len(instructionAccounts) != 1` fails, as there are two accounts passed in the instruction.
4. The error is propagated up to the `ObserveInbound` function, which re-scans the same transaction on the next ticker.
5. The error is surfaced repeatedly, preventing any other inbound Solana transactions from being processed.

## Impact

The Solana inbound processing is blocked indefinitely, preventing any other inbound Solana transactions from being processed.

## PoC

To demonstrate how an additional account breaks this check, we can run the[`TestSolanaToZEVMCall`](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/e2e/e2etests/test_solana_to_zevm_call.go) E2E test case. This test case calls the Solana gateway's `call` function with an additional account.

Update the [`CreateSOLCallInstruction`](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/e2e/runner/solana.go#L78-L99) function to add an additional account to the instruction:

```diff
return &solana.GenericInstruction{
		ProgID:    r.GatewayProgram,
		DataBytes: callData,
		AccountValues: []*solana.AccountMeta{
			solana.Meta(signer).WRITE().SIGNER(),
+			solana.Meta(solana.SystemProgramID),
		},
	}
```

Run the test case with the following command:

```bash
make start-solana-test
```

After a while, when the E2E test is running, you will see the following error in the Docker logs of the zetaclient (both, 1 and 2):

```bash
2025-05-07T12:21:46Z INF got wrong amount of signatures chain=902 chain_network=solana method=ObserveInbound module=inbound signatures=1

2025-05-07T12:21:46Z ERR task failed error="error FilterInboundEventAndVote for sig 2f4P2EAtrh7HitBEU4ZzvPCw72PcAR5EjqPqfTaqi5oyvFiY8zUwNdnvK9mxWN3kumEpnWWXCasbiNhPNkTD3cj: error FilterInboundEvent: error ParseInboundAsCall: want only 1 signer account, got 2" module=scheduler task.group=sol:902 task.name=observe_inbound task.type=interval_ticker
```

The error will be logged repeatedly as the zetaclients re-scan this same transaction. Any other inbound Solana transaction will not be processed.

## Mitigation

Like other instances in the Solana zetaclient, the check for the number of accounts should use `< 1` instead of `!= 1`. This will allow the program to process the transaction successfully, even if additional accounts are passed in the instruction.

```go
264: 	if len(instructionAccounts) < 1 {
265: 		return nil, fmt.Errorf("want required 1 signer account, got %d", len(instructionAccounts))
266: 	}
```


## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/zeta-chain/node/pull/3895


**gjaldon**

The issue is fixed by changing the [check](https://github.com/zeta-chain/node/pull/3895/files\#diff-ecd8b83a122aff6936f99a346430e981ae911aabd6a58d26dfaa75dd774d0617R248-R249) to less than the accountsNumberCall so that more instruction accounts can be passed.

---
### Example 2

**Auto Label:** Inconsistent state between data structures due to flawed traversal, mutation, or validation in linked lists, leading to incorrect ordering, infinite loops, or integer overflows.  

**Original Text Preview:**

## Context
(No context files were provided by the reviewer)

## Description/Recommendation
- **MechNvmSubscriptionNative.sol#L6-L18**: The interface `IERC1155` is defined but not used.
- **Karma.sol#L48**:
  - `external{` should be changed to `external {`
- **OlasMech.sol#L105-L117**: Here the code implicitly assumes that the previous element of the newly added link node is `0`:
  ```
  mapRequestIds[requestId][0] == 0 // the `0` sentinel node
  ```
    Currently, this assumption is correct for freshly added request ids and the ones that have been previously cleared out. But perhaps adding a comment here to mention this assumption might be useful.

## Status
- **Valory**: Fixed in commit `71b10e08`.
- **Cantina Managed**: Fix verified.

---
### Example 3

**Auto Label:** Inconsistent state between data structures due to flawed traversal, mutation, or validation in linked lists, leading to incorrect ordering, infinite loops, or integer overflows.  

**Original Text Preview:**

##### Description

Initializing variables to the default value executes an extra order that is not required.

Code Location
-------------

[FeeExecutor.sol#L294](https://github.com/primex-finance/primex_contracts/blob/f809cc0471935013699407dcd9eab63b60cd2e22/src/contracts/BonusExecutor/FeeExecutor.sol#L294)

#### FeeExecutor.sol

```
 uint256 lowest = 0;

```

[UniswapInterfaceMulticall.sol#L30](https://github.com/primex-finance/primex_contracts/blob/f809cc0471935013699407dcd9eab63b60cd2e22/src/contracts/UniswapInterfaceMulticall.sol#L30)

#### UniswapInterfaceMulticall.sol

```
for (uint256 i = 0; i < calls.length; i++) {

```

[LimitOrderLibrary.sol#L469](https://github.com/primex-finance/primex_contracts/blob/f809cc0471935013699407dcd9eab63b60cd2e22/src/contracts/libraries/LimitOrderLibrary.sol#L469)

#### LimitOrderLibrary.sol

```
for (uint256 i = 0; i < A.length - 1; i++) {

```

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:N/S:U (0.0)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:N/S:U)

##### Recommendation

Consider avoiding initializing variables to default value.

##### Remediation

**SOLVED**: The **Primex team** solved the issue by avoiding initializing variables to default value.

`Commit ID:` [694e59382fb9e378fecef1ef23af23b097c7bc10](https://github.com/primex-finance/primex_contracts/commit/694e59382fb9e378fecef1ef23af23b097c7bc10)

---
