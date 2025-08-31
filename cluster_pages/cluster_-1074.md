# Cluster -1074

**Rank:** #419  
**Count:** 10  

## Label
Improper validation of input parameters and transaction data leads to unauthorized function execution, incorrect state transitions, or bypassed access controls, enabling reentrancy, spoofed calls, or denial-of-service.

## Cluster Information
- **Total Findings:** 10

## Examples

### Example 1

**Auto Label:** Arbitrary function execution via unvalidated inputs, enabling attackers to invoke unintended operations, exfiltrate assets, or disrupt system logic through malformed or crafted transactions.  

**Original Text Preview:**

## Code Quality Issue

**Diﬃculty:** N/A

**Type:** Code Quality

## Description
The `CommandSafeExecutor` contract performs arbitrary calls into the compliance diamond, potentially allowing an unintentional function from the diamond to be called and trigger unexpected behavior, up to and including the drainage of native tokens from the margin account contract.

The compliance diamond implements two sets of contracts: the `WhitelistController` contract and those folding the various validator functions for each supported protocol. The validator functions for each protocol use the same ABI as the function the user intends to call on the targeted protocol.

### Intended Use Case

When the `CommandSafeExecutor` contract receives a transaction the user wants to call, such as:
```solidity
swapExactTokenForPt(address,address,uint256)
```
it calls the compliance contract using the same ABI and the same parameters the user has provided. The compliance facet of the diamond implements `swapExactTokenForPt(address,address,uint256)` and performs all of the intended validations, finally returning a `Command` struct that represents the sanitized version of the transaction. 

The `CommandSafeExecutor` contract then executes the sanitized command and includes a `msg.value` from the original user call.

### Issues Identified

The primary issue is that `CommandSafeExecutor` cannot determine whether a function being called on the compliance diamond is actually compliance-related. This means a user can call other facets of the compliance diamond, such as the functions implemented by `WhitelistController` or functions implemented by the diamond proxy.

Several functions implemented by `WhitelistController`, such as `getOperators`, return memory structures that could be manipulated into being decoded as a `Command[]`. If an attacker can manipulate the “target” field of the decoded command to an address they control, they can exfiltrate native token values due to how `execute()` uses the value specified in the original command, as shown in Figure 11.1. This is a similar mechanism to that described in TOB-ARK-10.

```solidity
function execute(Command calldata _cmd, bytes memory _encodedCmds) private {
    Command[] memory cmdsToExecute = abi.decode(_encodedCmds, (Command[]));
    cmdsToExecute.last().value = _cmd.value;
    cmdsToExecute.safeCallAll();
}
```

**Figure 11.1:** The `execute()` function uses the value specified by the command provided by the user. (contracts/marginAccount/base/CommandSafeExecutor.sol#30–35)

This finding would normally have a severity of “high,” but due to time constraints, we were unable to verify the finding’s exploitability and have left the severity undeclared.

The `abi.decode()` function will revert if a naive attempt is made to convert an encoded `string[] memory` to `Command[] memory`, but this conversion may still be technically possible given the right data structure or array contents.

## Exploit Scenario

A malicious user constructs a `Command` to be executed with its target set to the zero address, a payload mapping to the ABI of a function implemented by `WhitelistController`, and a value equal to all of the native tokens stored in the margin account.

The `CommandSafeExecutor` contract executes the `WhitelistController` function, which returns an encoded memory payload that will decode into a malicious `Command` payload that targets a malicious user-controlled contract.

The `CommandSafeExecutor` contract then sets the value on the command to be executed to the value in the original command, equal to all of the native tokens stored in the margin account.

When the command is executed, the fallback function of the malicious user’s contract is called, and the contract successfully receives the account’s native tokens.

## Recommendations

**Short term:** Modify the compliance-related functions to be gated behind a specific function such as `validatePayload(Command cmd)`. This function would iterate over all supported compliance functions and select the correct validator based on the command’s function signature. This would prevent unauthorized functions from being called by `CommandSafeExecutor` and provide a route where the `command.value` can be provided to the validator function.

**Long term:** Avoid patterns that allow contracts to call arbitrary function signatures. If the decision to use the diamond proxy pattern was based on needing the ability to arbitrarily add function validators to the diamond, it is recommended to remove the complexity introduced by the diamond proxy and move to a more traditional proxy standard to aid the project’s complexity management and maintainability.

---
### Example 2

**Auto Label:** Arbitrary function execution via unvalidated inputs, enabling attackers to invoke unintended operations, exfiltrate assets, or disrupt system logic through malformed or crafted transactions.  

**Original Text Preview:**

## Diﬃculty: N/A

## Type: Undeﬁned Behavior

### Description

The CommandSafeExecutor library is used to call into the compliance contract and validate user commands. However, it does not perform validation on the native tokens transferred by the user’s command. Instead, the CommandSafeExecutor library simply uses the value provided by the untrusted user-provided Command value and executes it, as shown in **Figure 10.1**. This lack of validation allows users to exfiltrate native tokens through specific protocol operators such as the Uniswap router.

```solidity
function execute(Command calldata _cmd, bytes memory _encodedCmds) private {
    Command[] memory cmdsToExecute = abi.decode(_encodedCmds, (Command[]));
    cmdsToExecute.last().value = _cmd.value;
    cmdsToExecute.safeCallAll();
}
```
*Figure 10.1*: The execute() function uses the value specified by the command provided by the user. (contracts/marginAccount/base/CommandSafeExecutor.sol#30–35)

The Uniswap router has a relatively unknown property where any tokens left in the router contract after a transaction ends can be stolen by any user. This does not impact users during normal operations because tokens transferred to the router are consumed by the swap and credited to the user initiating the swap. Many other automated market makers use the same design pattern in their router contracts.

If Arkis adds Uniswap or a similar protocol as an authorized operator, a malicious user may create a swap that intentionally leaves tokens behind in the router, allowing a follow-up transaction to steal the funds.

Note that even if Arkis does not support a vulnerable Uniswap-like protocol, users may still leverage this issue to grief Arkis by destroying the leverage tokens and transferring them to a contract that does not have a rescue mechanism. This attack would cost the user their collateral, so the risk introduced by griefing is directly related to the maximum leverage a user can obtain for a specific amount of collateral.

### Exploit Scenario

A malicious user opens an account with native ETH as its leverage token. This account is authorized to interact with the Uniswap router and has USDC as an authorized token. The user then constructs a multi-call with the three following transactions:

1. Call `marginAccount.execute()` with a command that has the following configuration:
   - **Target**: Uniswap router
   - **Value**: 1 wei
   - **Payload**: Call `swapExactETHForTokens`, swapping 1 wei of ETH for USDC.

2. Call `marginAccount.execute()` with a command that has the following configuration:
   - **Target**: Uniswap router
   - **Value**: `balanceOf(marginAccount)`
   - **Payload**: Call `swapExactTokensForTokens`, swapping USDC for WETH.

3. Call `router.swapExactETHForTokens`, swapping `balanceOf(router)` for USDC, sending the results to an attacker-controlled account.

The first transaction is to simply create an amount of ERC-20 tokens in the margin account that can be used in the ERC-20 for ERC-20 swap in the second transaction. The second transaction sends the entire account’s native token balance to the Uniswap router. Since it calls an ERC-20 for ERC-20 swap function, none of the native ETH in the router will be refunded. The last transaction is called outside the context of Arkis and is used to convert the native ETH in the Uniswap router into USDC that is sent to an attacker-controlled address.

### Recommendations

Short term, refactor the system’s compliance mechanism so that the command value transferred can be validated by the compliance functions. More precise recommendations in addition to long-term recommendations are documented in **TOB-ARK-11**.

---
### Example 3

**Auto Label:** Improper validation of input parameters and transaction data leads to unauthorized function execution, incorrect state transitions, or bypassed access controls, enabling reentrancy, spoofed calls, or denial-of-service.  

**Original Text Preview:**

##### Description
There is a check at line https://github.com/MetaLex-Tech/borg-core/blob/9074503d37cfa1d777ef16f6c88b84c98b4f54eb/src/borgCore.sol#L154 which passes if there is any calldata specified for the call. It will also pass if there is some value transferred together with the call, which can lead to bypassing the previous check (https://github.com/MetaLex-Tech/borg-core/blob/9074503d37cfa1d777ef16f6c88b84c98b4f54eb/src/borgCore.sol#L141) for native token transfers.

##### Recommendation
We recommend adding a check for the transferred value when the contract method is triggered.

---
