# Cluster -1135

**Rank:** #32  
**Count:** 605  

## Label
Insufficient input validation allows malformed or invalid hex strings to be processed, leading to incorrect address generation, unauthorized operations, and potential permanent system damage or misrouting.

## Cluster Information
- **Total Findings:** 605

## Examples

### Example 1

**Auto Label:** Insufficient input validation allows malicious actors to assign invalid addresses or exceed bounds, leading to unintended state changes, fund loss, or exploitable transaction flows.  

**Original Text Preview:**

When operations with address parameters are performed, it is crucial to ensure the address is not set to zero. Setting an address to zero is problematic because it has special burn/renounce semantics. This action should be handled by a separate function to prevent accidental loss of access during value or ownership transfers.

Throughout the codebase, there are multiple instances where operations are missing a zero address check:

* The [`_setMessenger(messengerType, dstDomainId, srcChainToken, srcChainMessenger)`](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/contracts/AdapterStore.sol#L34) operation within the contract `AdapterStore` in `AdapterStore.sol`.
* The [`_setMessenger(messengerTypes[i], dstDomainIds[i], srcChainTokens[i], srcChainMessengers[i])`](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/contracts/AdapterStore.sol#L52) operation within the contract `AdapterStore` in `AdapterStore.sol`.
* The [`_adapterStore`](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/contracts/libraries/OFTTransportAdapterWithStore.sol#L17) operation within the contract `OFTTransportAdapterWithStore` in `OFTTransportAdapterWithStore.sol`.
* The [`_setOftMessenger(token, messenger)`](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/contracts/SpokePool.sol#L363) operation within the contract `SpokePool` in `SpokePool.sol`.

Consider adding a zero address check before assigning a state variable.

***Update:** Acknowledged, not resolved. The team stated:*

> *After internal discussion, we think that zero-address checks here are a bit of overkill because all the functions mentioned are only callable by admin (except for `_adapterStore` case, but that's contract creation, and this contract can only be used within the system after admin action)*

---
### Example 2

**Auto Label:** Insufficient input validation allows malicious actors to assign invalid addresses or exceed bounds, leading to unintended state changes, fund loss, or exploitable transaction flows.  

**Original Text Preview:**

## Severity

**Impact:** Medium, gateway will still has it's old address

**Likelihood:** Medium, only occurs if the user decides to set a new address for his gateway

## Description

The gateway owner can set a new address for it using `setGatewayAddress` function. Unfortunately, this function only updates `gatewayByAddress` mapping, leaving `Gateway` struct intact

```solidity
  function setGatewayAddress(bytes calldata peerId, address newAddress) public {
    (Gateway storage gateway, bytes32 peerIdHash) = _getGateway(peerId);
    _requireOperator(gateway);

    if (gateway.ownAddress != address(0)) {
>>    delete gatewayByAddress[gateway.ownAddress];
    }

    if (address(newAddress) != address(0)) {
      require(gatewayByAddress[newAddress] == bytes32(0), "Gateway address already registered");
>>     gatewayByAddress[newAddress] = peerIdHash;
    }
```

If the user tries to call `allocateComputationUnits`, the contract will expect the old gateway address in `msg.sender`

```solidity
  function allocateComputationUnits(bytes calldata peerId, uint256[] calldata workerId, uint256[] calldata cus)
    external
    whenNotPaused
  {
    require(workerId.length == cus.length, "Length mismatch");
    (Gateway storage gateway,) = _getGateway(peerId);
>>  require(gateway.ownAddress == msg.sender, "Only gateway can allocate CUs");
```

Coded POC for `GatewayRegistry.unstake.t.sol`

```solidity
  function test_SetAddress() public {
    GatewayRegistry.Gateway memory gt = gatewayRegistry.getGateway(bytes("peerId"));
    bytes32 peerIdHash = gatewayRegistry.gatewayByAddress(address(this));
    // check previous gateway address
    assertEq(gt.ownAddress, address(this));
    assertEq(peerIdHash, keccak256("peerId"));
    // try set a new one
    address newAddy = address(0x6a7e);
    gatewayRegistry.setGatewayAddress(bytes("peerId"), newAddy);
    gt = gatewayRegistry.getGateway(bytes("peerId"));
    peerIdHash = gatewayRegistry.gatewayByAddress(newAddy);
    assertEq(peerIdHash, keccak256("peerId"));
    // this will fail, since address is not updated
    assertEq(gt.ownAddress, newAddy);
  }
```

## Recommendations

Update `ownAddress` variable as well

```solidity
    ...
    if (address(newAddress) != address(0)) {
      require(gatewayByAddress[newAddress] == bytes32(0), "Gateway address already registered");
      gatewayByAddress[newAddress] = peerIdHash;
>>    gateway.ownAddress = newAddress;
    }
```

---
### Example 3

**Auto Label:** Insufficient input validation allows malicious actors to assign invalid addresses or exceed bounds, leading to unintended state changes, fund loss, or exploitable transaction flows.  

**Original Text Preview:**

```solidity
function proRataRedeem(uint256 shares, address to, address from) public {
        // Effects
        uint16 feeBps = BasketManager(basketManager).managementFee(address(this));
        address feeCollector = BasketManager(basketManager).feeCollector();
        _harvestManagementFee(feeBps, feeCollector);
@>        if (msg.sender != from) {
            _spendAllowance(from, msg.sender, shares);
        }

        // Interactions
        BasketManager(basketManager).proRataRedeem(totalSupply(), shares, to);

        // We intentionally defer the `_burn()` operation until after the external call to
        // `BasketManager.proRataRedeem()` to prevent potential price manipulation via read-only reentrancy attacks. By
        // performing the external interaction before updating balances, we ensure that total supply and user balances
        // cannot be manipulated if a malicious contract attempts to reenter during the ERC20 transfer (e.g., through
        // ERC777 tokens or plugins with callbacks).
        _burn(from, shares);
    }
```

The operator has the authority to manage all funds. If msg.sender is the operator, they can perform the redeem operation without requiring any allowance.
Recommendation:
if (msg.sender != from) {
if (!isOperator[from][msg.sender]) {
\_spendAllowance(from, msg.sender, shares);
}
}

---
