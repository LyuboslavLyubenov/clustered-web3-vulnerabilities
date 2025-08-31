# Cluster -1339

**Rank:** #393  
**Count:** 13  

## Label
Failure to validate recipient addresses across chains leads to irreversible asset misdirection or manipulation due to incorrect address mapping or assumptions about address consistency.

## Cluster Information
- **Total Findings:** 13

## Examples

### Example 1

**Auto Label:** Failure to validate recipient addresses across chains leads to irreversible asset misdirection or manipulation due to incorrect address mapping or assumptions about address consistency.  

**Original Text Preview:**

`_quote` creates `_payload`, which consists of `shareAmount` and `destinationChainReceiver`, and passes it to `mailbox.quoteDispatch`.

```solidity
    function _quote(uint256 shareAmount, BridgeData calldata data) internal view override returns (uint256) {
>>>     bytes memory _payload = abi.encode(shareAmount, data.destinationChainReceiver);
        bytes32 msgRecipient = _addressToBytes32(selectorToChains[data.chainSelector].targetTeller);

        return mailbox.quoteDispatch(
            data.chainSelector, msgRecipient, _payload, StandardHookMetadata.overrideGasLimit(data.messageGas)
        );
    }
```

However, when the actual bridge is performed, the payload consists of `shareAmount`, `destinationChainReceiver`, and `messageId`.

```solidity
    function _bridge(uint256 shareAmount, BridgeData calldata data) internal override returns (bytes32 messageId) {
        // ...

>>>     bytes memory _payload = abi.encode(shareAmount, data.destinationChainReceiver, messageId);

        // Unlike L0 that has a built in peer check, this contract must
        // constrain the message recipient itself. We do this by our own
        // configuration.
        bytes32 msgRecipient = _addressToBytes32(selectorToChains[data.chainSelector].targetTeller);
        mailbox.dispatch{ value: msg.value }(
            data.chainSelector, // must be `destinationDomain` on hyperlane
            msgRecipient, // must be the teller address left-padded to bytes32
            _payload,
            StandardHookMetadata.overrideGasLimit(data.messageGas) // Sets the refund address to msg.sender, sets
                // `_msgValue`
                // to zero
        );
    }
```

Consider to also provide `messageId` inside `_quote`'s payload.

---
### Example 2

**Auto Label:** Failure to validate recipient addresses across chains leads to irreversible asset misdirection or manipulation due to incorrect address mapping or assumptions about address consistency.  

**Original Text Preview:**

**Description:** When a user bridges their YTokens using CCIP, they call [`BridgeCCIP::send`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/ccip/BridgeCCIP.sol#L117-L158). One of the parameters passed to this function is `_receiver`, which is intended to be the destination contract on the receiving chain:

```solidity
function send(address _yToken, uint64 _dstChain, address _to, uint256 _amount, address _receiver) external payable notBlacklisted(msg.sender) notBlacklisted(_to) notPaused {
    require(_amount > 0, "!amount");
    require(lockboxes[_yToken] != address(0), "!token !lockbox");
    require(IERC20(_yToken).balanceOf(msg.sender) >= _amount, "!balance");
    require(_to != address(0), "!receiver");
    require(tokens[_yToken][_dstChain] != address(0), "!destination");

    bytes memory _encodedMessage = abi.encode(_dstChain, _to, tokens[_yToken][_dstChain], _amount, Constants.BRIDGE_SEND_HASH);

    // Sends the message to the destination endpoint
    Client.EVM2AnyMessage memory evm2AnyMessage = Client.EVM2AnyMessage({
        // @audit-issue `_receiver` not verified
        receiver: abi.encode(_receiver), // ABI-encoded receiver address
        data: abi.encode(_encodedMessage), // ABI-encoded string
        tokenAmounts: new Client.EVMTokenAmount[](0), // Empty array indicating no tokens are being sent
        extraArgs: Client._argsToBytes(Client.EVMExtraArgsV2({ gasLimit: 200_000, allowOutOfOrderExecution: true })),
        feeToken: address(0) // For msg.value
    });
```

However, the `_receiver` parameter is not validated. If the user provides an incorrect or malicious address, the message may be delivered to a contract that cannot handle it, resulting in unrecoverable loss of the bridged tokens.

**Recommended Mitigation:** Validate the `_receiver` address against a trusted mapping, such as the `peers` mapping mentioned in a previous finding, to ensure it corresponds to a legitimate contract on the destination chain.

**YieldFi:** Fixed in commit [`a03341d`](https://github.com/YieldFiLabs/contracts/commit/a03341d8103ba08473ea1cd39e64192608692aca)

**Cyfrin:** Verified. `_receiver ` is now verified to be a trusted peer.

---
### Example 3

**Auto Label:** Failure to validate recipient addresses across chains leads to irreversible asset misdirection or manipulation due to incorrect address mapping or assumptions about address consistency.  

**Original Text Preview:**

**Description:** When sending cross-chain messages via CCIP, Chainlink recommends keeping the `extraArgs` parameter mutable to allow for future upgrades or configuration changes, as outlined in their [best practices](https://docs.chain.link/ccip/best-practices#using-extraargs).

However, this recommendation is not followed in [`BridgeCCIP::send`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/bridge/ccip/BridgeCCIP.sol#L126-L133), where `extraArgs` is hardcoded:
```solidity
// Sends the message to the destination endpoint
Client.EVM2AnyMessage memory evm2AnyMessage = Client.EVM2AnyMessage({
    receiver: abi.encode(_receiver), // ABI-encoded receiver address
    data: abi.encode(_encodedMessage), // ABI-encoded string
    tokenAmounts: new Client.EVMTokenAmount[](0), // Empty array indicating no tokens are being sent
    // @audit-issue `extraArgs` hardcoded
    extraArgs: Client._argsToBytes(Client.EVMExtraArgsV2({ gasLimit: 200_000, allowOutOfOrderExecution: true })),
    feeToken: address(0) // For msg.value
});
```

**Impact:** Because `extraArgs` is hardcoded, any future changes would require deploying a new version of the bridge contract.

**Recommended Mitigation:** Consider making `extraArgs` mutable by either passing it as a parameter to the `send` function or deriving it from configurable contract storage.

**YieldFi:** Fixed in commits [`3cc0b23`](https://github.com/YieldFiLabs/contracts/commit/3cc0b2331c35327a43e95176ce6c5578f145c0ee) and [`fd4b7ab5`](https://github.com/YieldFiLabs/contracts/commit/fd4b7ab57a5ae2ac366b4d9d086eb372defc7f8c)

**Cyfrin:** Verified. `extraArgs` is now passed as a parameter to the call.

---
