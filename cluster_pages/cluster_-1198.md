# Cluster -1198

**Rank:** #390  
**Count:** 13  

## Label
Failure to validate `msg.sender` or transaction success leads to unauthorized access, incorrect state updates, or fund loss through improper authorization or reentrancy-like behavior.

## Cluster Information
- **Total Findings:** 13

## Examples

### Example 1

**Auto Label:** Failure to validate `msg.sender` or transaction success leads to unauthorized access, incorrect state updates, or fund loss through improper authorization or reentrancy-like behavior.  

**Original Text Preview:**

<https://github.com/code-423n4/2024-07-benddao/blob/main/src/yield/YieldStakingBase.sol#L312><br><https://github.com/code-423n4/2024-07-benddao/blob/main/src/yield/YieldStakingBase.sol#L384>

### Impact

When `botAdmin` attempts to unstake and repay risky positions, wrong account address will be used, causing the transaction to fail.

### Proof of Concept

In the `YieldStakingBase.sol` contract, `unstake` and `repay` transaction can only be initiated by either the NFT owner or `botAdmin`. Unfortunately, there is a bug in which `msg.sender` is used to fetch borrower yield account address from the `yieldAccounts` mapping:

```solidity
  function _unstake(uint32 poolId, address nft, uint256 tokenId, uint256 unstakeFine) internal virtual {
    UnstakeLocalVars memory vars;

>>  vars.yieldAccout = IYieldAccount(yieldAccounts[msg.sender]);
    require(address(vars.yieldAccout) != address(0), Errors.YIELD_ACCOUNT_NOT_EXIST);

  function _repay(uint32 poolId, address nft, uint256 tokenId) internal virtual {
    RepayLocalVars memory vars;

>>  vars.yieldAccout = IYieldAccount(yieldAccounts[msg.sender]);
    require(address(vars.yieldAccout) != address(0), Errors.YIELD_ACCOUNT_NOT_EXIST);
```

If `botAdmin` calls one of these functions, the protocol will attempt to use `botAdmin` yield account address instead of the borrower account, resulting in incorrect account data being used and transaction failure. Since `botAdmin` is the only address that can force close risky positions, this bug poses a high threat to the protocol.

### Recommended Mitigation Steps

Allow specifying `user` address in `unstake` and `repay` functions:

```diff
+ function _unstake(uint32 poolId, address nft, address user, uint256 tokenId, uint256 unstakeFine) internal virtual {
    UnstakeLocalVars memory vars;

+   vars.yieldAccout = IYieldAccount(yieldAccounts[user]);
+   require(user == msg.sender || botAdmin == msg.sender, 
Errors.INVALID_CALLER);
    require(address(vars.yieldAccout) != address(0), Errors.YIELD_ACCOUNT_NOT_EXIST);
```

**[thorseldon (BendDAO) confirmed and commented](https://github.com/code-423n4/2024-07-benddao-findings/issues/6#issuecomment-2316805726):**
> Fixed [here](https://github.com/BendDAO/bend-v2/commit/bb2edf049c2efe20a78fdabefb9a3a45e3dcaf67).

***

---
### Example 2

**Auto Label:** Failure to validate message origin or caller identity leads to unauthorized asset transfers or privilege escalation via dangling allowances or improper sender checks.  

**Original Text Preview:**

##### Description
- https://github.com/LayerZero-Labs/LayerZero-v2/blob/592625b9e5967643853476445ffe0e777360b906/packages/layerzero-v2/evm/oapp/contracts/oft/OFTCore.sol#L211
- https://github.com/defidotmoney/dfm-core/blob/0bde9fb784400d73f434d585aab7d9dd0ad1c679/contracts/bridge/BridgeToken.sol#L153

The `OFTCore.send()` method calls `msgInspector`. However, the new `BridgeToken.sendSimple()` method does not involve such a call.

Thus, if `msgInspector` is configured, the call to `msgInspector` will be ignored when `sendSimple()` is called.

##### Recommendation
We recommend that `msgInspector` is also called in `sendSimple()`.

---
### Example 3

**Auto Label:** Failure to validate `msg.sender` or transaction success leads to unauthorized access, incorrect state updates, or fund loss through improper authorization or reentrancy-like behavior.  

**Original Text Preview:**

##### Description

The ability of the handler to accept payments through any user using the receive() function in Solidity in this system could potentially result in loss of funds. To prevent sending funds through the receive() and fallback() functions in Solidity, you can implement checks and safeguards in your contract to ensure that only payments from specific approved contracts are accepted. This can be done by specifying a list of approved contracts that are allowed to send funds to your contract, and by implementing appropriate checks to verify that the sender of the payment is one of these approved contracts. For example, you can use the msg.sender variable in Solidity to check the sender of the payment, and only accept payments from the approved contracts. You can also use the require() function to enforce these checks and prevent payments from unauthorized contracts from being accepted. It is important to carefully review and test your contract to ensure that it properly implements these checks and safeguards.

Code Location
-------------

[NativeHandler.sol#L187](https://gitlab.mediarex.com/mediarex/blockchain/bridge-contracts/-/blob/094856fd48bb4627842c72e914df09fe9689ad81/contracts/handlers/NativeHandler.sol#L187)

```
    receive() external payable { }

```

![poc5.jpg](https://halbornmainframe.com/proxy/audits/images/659ec8b2a1aa3698c0ee4237)

```
contract Tester is DSTest {

     address public user1 = address(0xB0B);
     Bridge internal bridge;
     NativeHandler internal nativeHandler;
     CheatCodes cheats = CheatCodes(0x7109709ECfa91a80626fF3989D68f67F5b1DD12D);
   function setUp() public {
        address [] memory path = new address[](2);
        path[0] = address(this);
        path[1] = address(this);
        bridge = new Bridge(1,path,5,100,100);

        bytes32 [] memory pathByteGG = new bytes32[](2);
        pathByteGG[0] = keccak256(abi.encodePacked("User1"));
        pathByteGG[1] = keccak256(abi.encodePacked("User2"));

        nativeHandler = new NativeHandler(address(bridge),pathByteGG,path,path);

        bytes32 tokenResource = keccak256(abi.encodePacked("NATIVE"));
        bridge.adminSetResource(address(nativeHandler),tokenResource,0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE);

        cheats.deal(user1,1_000_000 ether);
        cheats.deal(address(this), 1_000_000 ether);
   }

   function testHandlerDepositDirectly() public {
        address(nativeHandler).transfer(5000 ether);
   }
}

```

##### Score

Impact: 5  
Likelihood: 3

##### Recommendation

**SOLVED**: The `Chiliz team` solved the issue by adding a necessary check in the handler.

`Commit ID:` [49e3c75605632664540b946df8bda7892fc7e884](https://gitlab.mediarex.com/mediarex/blockchain/bridge-contracts/-/commit/49e3c75605632664540b946df8bda7892fc7e884)

---
