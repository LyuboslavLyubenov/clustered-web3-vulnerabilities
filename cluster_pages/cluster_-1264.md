# Cluster -1264

**Rank:** #201  
**Count:** 55  

## Label
Race conditions and improper state updates lead to denial of service, fund loss, or broken rollover flows due to flawed indexing, premature checks, or stale references in critical system operations.

## Cluster Information
- **Total Findings:** 55

## Examples

### Example 1

**Auto Label:** Race conditions and improper state updates lead to denial of service, fund loss, or broken rollover flows due to flawed indexing, premature checks, or stale references in critical system operations.  

**Original Text Preview:**

`ZipperFactoryKey` is used to assign `keyToUser`, where we can also update the old `keyToUser`, by assigning a new user:

```solidity
  function changeKey(uint16 chainId, address user, bytes memory key) external onlyRole(CHANGE_KEY_ROLE) {
    require(chain.validateEncodedAddress(chainId, key), "Invalid key");

    bytes memory oldKey = userToKey[chainId][user];
    require(oldKey.length > 0, "Key does not exist for user");
    require(keccak256(oldKey) != keccak256(key), "Key is the same");

    userToKey[chainId][user] = key;
    keyToUser[chainId][key] = user;
```

This issue we have here is that `user` at `keyToUser[chainId][oldKey]` is never removed, meaning even after changing the `userToKey` to a new key and `keyToUser` to a new user, the map `keyToUser[chainId][oldKey]` still contain the old `user`.


This map is used when requesting a deposit:

```solidity
  function reqDepositZToken( ... ) external nonReentrant onlyRole(RELAYER_ROLE) {
    // ...

    address user = keyFactory.keyToUser(chainId, receiver);

    // ...

    Request memory request = Request({
      reqType: REQ_DEPOSIT_ZTOKEN,
      chainId: chainId,
      // NOTE srcHashTx are enought to be unique identifiers
      payload: abi.encode(zToken, sender, receiver, user, zAmount, srcHashTx, srcHashIdx)
    });
```

Where it's extremely dangerous as this user is the one receiving the funds when we `executeRequest`:

```solidity
  function executeRequest(uint256 _requestId) external nonReentrant onlyRole(EXECUTOR_ROLE) {
    // ...
    if (request.reqType == REQ_DEPOSIT_ZTOKEN) {
      (address zToken, , , address user, uint256 zAmount, , ) = abi.decode(
        request.payload,
        (address, bytes, bytes, address, uint256, bytes, uint256)
      );
      tokenFactory.mintZToken(request.chainId, zToken, user, zAmount);
```

In short if the old key is entered, then the tokens will be transferred to the old user, which can be a no longer used address or abandoned contract.

Note that it's the same in `ZipperFactoryVault::changeVault`, but the impact there is non-existent.

Recommendations:

Remove the old key from `keyToUser`:
```diff
    userToKey[chainId][user] = key;
    keyToUser[chainId][key] = user;
+   delete keyToUser[chainId][oldKey];
```

---
### Example 2

**Auto Label:** Race conditions and improper state updates lead to denial of service, fund loss, or broken rollover flows due to flawed indexing, premature checks, or stale references in critical system operations.  

**Original Text Preview:**

## Security Audit Report

## Function
### Severity
High Risk

### Context
`LockingController.sol#L253`

### Description
LockedPositionTokens are transferable unrestricted until used to vote. This allows holders to perform blind transfers to the gateway contract. The above capability coupled with the `LockingController.increaseUnwindingEpochs` use of `msg.sender`'s balance means that the existence of any prior transfers to the gateway result in attempting to burnFrom the gateway an amount larger than it will ever approve.

### Proof of Concept

#### Scenario:
- Alice creates a position.
- Bob creates a position.
- Bob sends his position to the gateway.
- Alice attempts to `increaseUnwindingEpochs` and reverts.

The revert is due to the function attempting to `burnFrom` the gateway both her position and Bob's transferred tokens. 

Below is a modified version of `testIncreaseUnwindingEpochs` which now reverts with the addition of Bob's transfer.

```solidity
function testDosIncreaseUnwindingEpochs() public {
    _createPosition(alice, 1000, 10);
    _createPosition(bob, 1000, 10);
    LockedPositionToken liusd = LockedPositionToken(lockingController.shareToken(10));
    
    // Mirrors `testIncreaseUnwindingEpochs` test with the only difference being Bob transfers to the gateway.
    vm.prank(bob);
    liusd.transfer(address(gateway), 1000);
    
    assertEq(
        lockingController.balanceOf(alice),
        1000,
        "Error: Alice's balance after creating first position is not correct"
    );
    assertEq(
        lockingController.rewardWeight(alice),
        1200,
        "Error: Alice's reward weight after creating first position is not correct"
    );
    assertEq(
        lockingController.shares(alice, 10),
        1000,
        "Error: Alice's share after creating first position is not correct"
    );
    assertEq(
        lockingController.shares(alice, 12),
        0,
        "Error: Alice's share after increasing unwinding epochs is not correct"
    );

    vm.startPrank(alice);
    {
        MockERC20(lockingController.shareToken(10)).approve(address(gateway), 1000);
        gateway.increaseUnwindingEpochs(10, 12);
    }
    vm.stopPrank();

    assertEq(
        lockingController.balanceOf(alice),
        1000,
        "Error: Alice's balance after increasing unwinding epochs is not correct"
    ); // unchanged
    assertEq(
        lockingController.rewardWeight(alice),
        1240,
        "Error: Alice's reward weight after increasing unwinding epochs is not correct"
    ); // +40
    assertEq(
        lockingController.shares(alice, 10),
        0,
        "Error: Alice's share after increasing unwinding epochs is not correct"
    ); // -1000
    assertEq(
        lockingController.shares(alice, 12),
        1000,
        "Error: Alice's share after increasing unwinding epochs is not correct"
    ); // +1000
}
```

### Recommendation
In the gateway, make use of the shares variable `uint256 shares = liusd.balanceOf(msg.sender);` and pass as an argument into `LockingController.increaseUnwindingEpochs`. This requires an interface edit to `LockingController` as well.

### Acknowledgments
- **infiniFi**: Fixed in 708f8bf.
- **Spearbit**: Fix verified.

---
### Example 3

**Auto Label:** Race conditions and improper state updates lead to denial of service, fund loss, or broken rollover flows due to flawed indexing, premature checks, or stale references in critical system operations.  

**Original Text Preview:**

The module allows a registered chainlink upkeep to automatically re-lock aura locks

Funds cannot be stolen in any way

Due to the lax timing and lack of MEV, I cannot expect the upkeep to cause any particular issue

It's worth noting that in case you want to deprecate the module, for example to stop re-locking, it will be sufficient to remove it from the safe modules

There is no particular risk tied to adding this module as in the worst case it will consume a bit of `LINK` token to perform the upkeep

---
