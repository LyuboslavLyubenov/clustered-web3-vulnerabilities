# Cluster -1212

**Rank:** #168  
**Count:** 75  

## Label
Inconsistent state validation and access controls across contracts lead to unauthorized access, state mismatches, and exploitable behavior due to missing or misaligned parameter checks and invariant enforcement.

## Cluster Information
- **Total Findings:** 75

## Examples

### Example 1

**Auto Label:** **Improper state transitions and lifecycle management lead to unauthorized access, reentrancy, and data leakage via residual connections or unvalidated interactions.**  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/851 

## Found by 
0xDgiin, 0xEkko, 0xbakeng, 0xgee, 4n0nx, Aenovir, PNS, Tychai0s, dystopia, future, gkrastenov, lazyrams352, patitonar, pv, realsungs, theweb3mechanic, xiaoming90

## Summary
The failure to lock collateral in `borrowCrossChain` before sending cross-chain borrow requests via LayerZero V2 enables users to initiate multiple borrow requests on different chains (e.g., Chain B and Chain C) using the same collateral on Chain A, resulting in loans exceeding the collateral’s capacity and significant protocol loss.

## Root Cause
In `CrossChainRouter.sol` within the `borrowCrossChain` function, the collateral amount is calculated but not locked in `lendStorage` before sending a borrow request via `_lzSend`. The collateral is only registered in `_handleValidBorrowRequest` after receiving confirmation from the destination chain, allowing multiple requests to use the same collateral concurrently.

[CrossChainRouter.borrowCrossChain](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/713372a1ccd8090ead836ca6b1acf92e97de4679/Lend-V2/src/LayerZero/CrossChainRouter.sol#L113-L113)

## Internal Pre-conditions
1. The user needs to have sufficient collateral on Chain A (e.g., $1000 USDC in an lToken pool).
2. The protocol needs to support cross-chain borrowing on at least two destination chains (e.g., Chain B and Chain C).
3. The lToken pools on destination chains need to have sufficient liquidity to process borrow requests.
4. The collateral factor (e.g., 75%) needs to allow significant borrowing relative to the collateral.

## External Pre-conditions
1. The underlying token prices (e.g., USDC) need to remain stable to ensure consistent collateral and borrow valuations.
2. The LayerZero V2 network needs to process messages with minimal delay to allow rapid request submission.

## Attack Path
1. A user with 1000 USDC collateral on Chain A calls `borrowCrossChain` twice in the same block:
   - Request 1: Borrow 750 USDC on Chain B.
   - Request 2: Borrow 750 USDC on Chain C.
2. Both requests use `getHypotheticalAccountLiquidityCollateral`, which reports 1000 USDC collateral, as no borrow is yet registered.
3. Chain B receives the request, verifies `payload.collateral` (1000 USDC) >= 750 USDC, and executes `borrowForCrossChain`, transferring 750 USDC to the user.
4. Chain C does the same, transferring another 750 USDC.
5. Both chains send `ValidBorrowRequest` messages back to Chain A, which registers two borrows totaling 1500 USDC in `_handleValidBorrowRequest`.
6. User does not repay the debt and sacrifices the collateral, leaving with $500 in profit.

## Impact
The protocol suffers a significant financial loss by granting loans exceeding the collateral’s capacity (e.g., $750 loss for 1500 USDC borrowed against 1000 USDC collateral). The issue scales with the number of chains, potentially leading to catastrophic losses (e.g., 7500 USDC for 10 chains).

## PoC
The issue can be demonstrated as follows:
- Deploy Lend-V2 on three chains: Chain A (collateral), Chain B, and Chain C (borrowing). A user supplies 1000 USDC collateral on Chain A in an lToken pool with a 75% collateral factor.
- The user calls `borrowCrossChain` twice in one block:
  - Chain B: 750 USDC borrow.
  - Chain C: 750 USDC borrow.
- `getHypotheticalAccountLiquidityCollateral` returns 1000 USDC collateral for both requests, as no borrow is registered.
- Chain B verifies `payload.collateral` (1000 USDC) >= 750 USDC, executes `borrowForCrossChain`, and transfers 750 USDC.
- Chain C does the same, transferring 750 USDC.
- Chain A receives `ValidBorrowRequest` messages, registering 1500 USDC total borrow.
- For a $10,000 collateral, the loss is $7500 across two chains (75% loss).

## Mitigation
Lock the collateral in `lendStorage` before sending the borrow request in `borrowCrossChain` by tracking pending borrow requests and deducting their collateral impact. Use a nonce-based lock to prevent concurrent requests for the same collateral. 
In `_handleValidBorrowRequest`, clear the pending borrow lock after registering the borrow. Add a check in `borrowCrossChain` to prevent new requests if a pending borrow exists for the same user and collateral.

---
### Example 2

**Auto Label:** Misleading function behavior and inadequate state handling lead to security risks, excessive gas, and user misinterpretation through flawed interface semantics and redundant operations.  

**Original Text Preview:**

**Description:** In the [pull request](https://github.com/YieldFiLabs/contracts/pull/19), the function [`YToken::_withdraw`](https://github.com/YieldFiLabs/contracts/blob/702a931df3adb2f6e48807203cdc7a92604ea249/contracts/core/tokens/YToken.sol#L193) was updated to be declared `virtual`, allowing it to be overridden in derived contracts. However, it is never actually overridden in any of the `dYToken` implementations.

Consider removing the `virtual` modifier from both `YToken::_withdraw` and `YTokenL2::_withdraw` to clarify intent and avoid misleading extensibility.

**YieldFi:** Acknowledged.

\clearpage

---
### Example 3

**Auto Label:** Inadequate access control and state validation create exploitable windows, enabling unauthorized state modifications, front-running, and deadlocks—compromising security, transparency, and decentralization.  

**Original Text Preview:**

<https://github.com/initia-labs/minievm/blob/744563dc6a642f054b4543db008df22664e4c125/x/evm/keeper/context.go# L388-L401>

### Finding description and impact

The `Create` and `Create2` message handlers check the sender is allowed to deploy a contract by calling `assertAllowedPublishers(..)` [here](https://github.com/initia-labs/minievm/blob/744563dc6a642f054b4543db008df22664e4c125/x/evm/keeper/msg_server.go# L51) and [here](https://github.com/initia-labs/minievm/blob/744563dc6a642f054b4543db008df22664e4c125/x/evm/keeper/msg_server.go# L89).

The deployers can [either be restricted to specific addresses (`params.AllowedPublishers`) or completely unrestricted](https://github.com/initia-labs/minievm/blob/744563dc6a642f054b4543db008df22664e4c125/x/evm/keeper/msg_server.go# L260-L283) so that anyone can deploy a contract.

However, if the deployers are restricted, a permissioned deployer can deploy a factory contract that can then be used by anyone to deploy arbitrary contracts, bypassing the restriction and potentially leading to malicious contracts being deployed.

This is contrary to Evmos which does check also the EVM `create` opcode if the address is allowed to deploy a contract. It [adds a special “create” hook](https://github.com/evmos/evmos/blob/0402a01c9036b40a7f47dc2fc5fb4cfae019f5f1/x/evm/keeper/state_transition.go# L65-L67), which [checks](https://github.com/evmos/evmos/blob/0402a01c9036b40a7f47dc2fc5fb4cfae019f5f1/x/evm/types/permissions.go# L67-L74) if the caller is permissioned to deploy a contract.

This hook is called by [`EVM::Create`](https://github.com/evmos/evmos/blob/0402a01c9036b40a7f47dc2fc5fb4cfae019f5f1/x/evm/core/vm/evm.go# L529-L531) and [`EVM::Create2`](https://github.com/evmos/evmos/blob/0402a01c9036b40a7f47dc2fc5fb4cfae019f5f1/x/evm/core/vm/evm.go# L541-L543), which are called when the EVM encounters the `CREATE` and `CREATE2` opcodes.

This shows that contract deployment can be fully restricted if desired.

### Proof of Concept

`EVMCreateWithTracer()` internally calls the EVM’s `Create` or `Create2` functions, which contain the standard implementation without any deployment restriction, found [here](https://github.com/initia-labs/evm/blob/3d312736d7fbf68af35df985b01536684bf6aafa/core/vm/evm.go# L569-L583).

### Recommended mitigation steps

Consider using a similar approach to Evmos whereby the `Create` and `Create2` are intercepted and the caller is checked for permission to deploy a contract.

**[leojin (Initia) disputed and commented via duplicate issue S-161](https://code4rena.com/evaluate/2025-02-initia-cosmos/submissions/S-161?commentParent=87nQMpezhRq):**

> Since the allowed publisher list is typically set at chain launch, this does not seem to be an issue that requires fixing. It also appears fine since the update parameters are controlled by an operator.

---

---
