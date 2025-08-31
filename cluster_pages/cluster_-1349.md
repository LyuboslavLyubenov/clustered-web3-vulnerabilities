# Cluster -1349

**Rank:** #425  
**Count:** 10  

## Label
Cross-chain message validation failures enabling unauthorized access, sender spoofing, or message manipulation, leading to denial-of-service, asset lock, or unintended state changes.

## Cluster Information
- **Total Findings:** 10

## Examples

### Example 1

**Auto Label:** Cross-chain message validation failures enabling unauthorized access, sender spoofing, or message manipulation, leading to denial-of-service, asset lock, or unintended state changes.  

**Original Text Preview:**

## LayerZeroAdapter.sol#L77

## Description
The `debug_forceReceiveMessage` function in `LayerZeroAdapter` is a restricted function that allows privileged actors to deliver arbitrary messages to the Bridge contract. For example, a privileged actor could deliver a random payload to the Bridge contract without sending it through LayerZero. This ability could lead to stealing NFTs from the Bridge contract and minting them without constraints. This also undermines the security offered by LayerZero as a cross-chain communication protocol.

## Recommendation
Consider removing the `debug_forceReceiveMessage` function.

## Sweep n' Flip
Fixed in `snf-bridge-contracts-v1 PR 15`.

## Cantina Managed
Verified fix.

---
### Example 2

**Auto Label:** Failure to validate VAA recipient or payload content enables arbitrary cross-chain execution or unauthorized actions, leading to potential misuse or exploitation of smart contracts.  

**Original Text Preview:**

##### Description

The `executeProposal()` function in the current smart contract is responsible for parsing and verifying VAAs (Validators Aggregated Attestations) and then executing transactions based on these VAAs. The function does not verify the `Chain ID` or the `receiver address` (recipient of the transaction).

The absence of chain ID and receiver address verification could lead to significant security issues. Since the chain ID and recipient address are not checked, an attacker can craft a VAA to target an address on another chain, causing a cross-chain replay attack.

* **Code Location**

```
   function _executeProposal(bytes memory VAA, bool overrideDelay) private {
        // This call accepts single VAAs and headless VAAs
        (
            IWormhole.VM memory vm,
            bool valid,
            string memory reason
        ) = wormholeBridge.parseAndVerifyVM(VAA);

        require(valid, reason); /// ensure VAA parsing verification succeeded

        if (!overrideDelay) {
            require(
                queuedTransactions[vm.hash].queueTime != 0,
                "TemporalGovernor: tx not queued"
            );
            require(
                queuedTransactions[vm.hash].queueTime + proposalDelay <=
                    block.timestamp,
                "TemporalGovernor: timelock not finished"
            );
        } else if (queuedTransactions[vm.hash].queueTime == 0) {
            /// if queue time is 0 due to fast track execution, set it to current block timestamp
            queuedTransactions[vm.hash].queueTime = block.timestamp.toUint248();
        }



        require(
            !queuedTransactions[vm.hash].executed,
            "TemporalGovernor: tx already executed"
        );

        queuedTransactions[vm.hash].executed = true;

        address[] memory targets; /// contracts to call
        uint256[] memory values; /// native token amount to send
        bytes[] memory calldatas; /// calldata to send
        (, targets, values, calldatas) = abi.decode(
            vm.payload,
            (address, address[], uint256[], bytes[])
        );

        /// Interaction (s)

        _sanityCheckPayload(targets, values, calldatas);

        for (uint256 i = 0; i < targets.length; i++) {
            address target = targets[i];
            uint256 value = values[i];
            bytes memory data = calldatas[i];

            // Go make our call, and if it is not successful revert with the error bubbling up
            (bool success, bytes memory returnData) = target.call{value: value}(
                data
            );

            /// revert on failure with error message if any
            require(success, string(returnData));

            emit ExecutedTransaction(target, value, data);
        }
    }

```

##### Proof of Concept

`Step 1 :` An attacker crafts a wormhole message that appears to be valid but is intended for a different chain (different chain ID) or is directed to an unintended recipient address.

`Step 2 :` The attacker submits this crafted payload to the `_executeProposal()` function in the smart contract.

`Step 3 :` Since there are no checks in place for the chain ID or recipient address, the function treats the VAA as valid and begins to execute the transaction(s) specified in the VAA payload.

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:H/A:H/D:N/Y:N/R:P/S:C (5.9)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:H/A:H/D:N/Y:N/R:P/S:C)

##### Recommendation

It is recommended to add the necessary validations.

  

### Remediation Plan

**SOLVED**: The `Moonwell Finance team` solved the issue by adding the necessary validations.

`Commit ID:` [c39f98bdc9dd4e448ba585923034af1d47f74dfa](https://github.com/moonwell-fi/moonwell-contracts-v2/commit/c39f98bdc9dd4e448ba585923034af1d47f74dfa)

##### Remediation Hash

<https://github.com/moonwell-fi/moonwell-contracts-v2/commit/c39f98bdc9dd4e448ba585923034af1d47f74dfa>

---
### Example 3

**Auto Label:** Cross-chain message validation failures enabling unauthorized access, sender spoofing, or message manipulation, leading to denial-of-service, asset lock, or unintended state changes.  

**Original Text Preview:**

## BridgedGovernor.sol Vulnerability Report

## Context
BridgedGovernor.sol#L57-L71

## Description
The `lzReceive` logic in `BridgedGovernor.sol` does not check the address of the executor.

```solidity
// BridgedGovernor.sol
function lzReceive(
    Origin calldata origin,
    bytes32, /* guid */
    bytes calldata message,
    address, /* executor */
    bytes calldata /* extraData */
) public payable onlyProxy {
    require(msg.sender == endpoint, "Must be called by the endpoint");
    require(origin.srcEid == ownerEid, "Invalid message source chain");
    require(origin.sender == owner, "Invalid message sender");
    require(origin.nonce == _lastNonce + 1, "Invalid message nonce");
    // slither-disable-next-line events-maths
    _lastNonce = origin.nonce;
    runCalls(abi.decode(message, (Call[])));
}
```

The `EndpointV2.lzReceive` function does not implement access control and can be called by anyone once the message is verified. This creates a potential vulnerability where a malicious actor can front-run the executor and invoke `lzReceive` with a different `msg.value` than the sender intended. Depending on the encoded message data, this could lead to a successful message delivery with a state update that differs from what was intended.

### Impact
**Medium/High**: This vulnerability can result in the loss of tokens used to cover executor options on the sending chain, along with unintended state updates in one of the external calls.

### Likelihood
**Low**: The `msg.value` is checked for each external call, which reduces the risk of exploitation.

### Recommendation
To mitigate this issue, it is recommended to encode the `msg.value` within the message on the sending chain, decode it in `lzReceive`, and compare it against the actual `msg.value`:

```solidity
// Suggested changes
- runCalls(abi.decode(message, (Call[])));
+ (uint256 msgValue, Call[] memory calls) = abi.decode(message, (uint256, Call[]));
+ require(msg.value >= msgValue, "Invalid message value");
+ runCalls(calls);
```

## Drips
This issue was fixed in commit `6a7b6c6a` by implementing the auditor's recommendations.

## Cantina Managed
This vulnerability has been addressed and fixed.

---
