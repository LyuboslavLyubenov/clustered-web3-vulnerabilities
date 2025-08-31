# Cluster -1351

**Rank:** #394  
**Count:** 13  

## Label
Insufficient ownership and authorization checks enable unauthorized asset manipulation, leading to loss of control, unauthorized transfers, and potential theft through improper validation of caller identity or transaction order.

## Cluster Information
- **Total Findings:** 13

## Examples

### Example 1

**Auto Label:** Insufficient ownership and authorization checks enable unauthorized asset manipulation, leading to loss of control, unauthorized transfers, and potential theft through improper validation of caller identity or transaction order.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-10-debita-judging/issues/890 

## Found by 
0x37, 0xPhantom2, 4lifemen, Audinarey, BengalCatBalu, CL001, Cybrid, DenTonylifer, Greed, Greese, IzuMan, KiroBrejka, KungFuPanda, Pro\_King, Valy001, alexbabits, araj, dhank, dimulski, durov, kazan, lanrebayode77, merlin, newspacexyz, nikhilx0111, pashap9990, shaflow01, t.aksoy, utsav, xiaoming90, ydlee
### Summary

After sellNFT is completed, the NFT should be transferred  to the order creator, but this is not done.

### Root Cause

After the buyOrder is completed, the order creator does not receive the NFT, and the NFT is sent directly to **[buyOrderContract](https://github.com/sherlock-audit/2024-11-debita-finance-v3/blob/1465ba6884c4cc44f7fc28e51f792db346ab1e33/Debita-V3-Contracts/contracts/buyOrders/buyOrder.sol#L99)**

The latter only emits an event and deletes the order, but does not transfer the NFT to  the order creator

### Internal pre-conditions



### External pre-conditions

1. User A  create buyOrder.
2. User B  **sellNFT**.

### Attack Path
1. User A  create buyOrder.
2. User B  **sellNFT**,and [receive](https://github.com/sherlock-audit/2024-11-debita-finance-v3/blob/1465ba6884c4cc44f7fc28e51f792db346ab1e33/Debita-V3-Contracts/contracts/buyOrders/buyOrder.sol#L122) **buyToken**
3.  But **order creator** will[ lose ](https://github.com/sherlock-audit/2024-11-debita-finance-v3/blob/1465ba6884c4cc44f7fc28e51f792db346ab1e33/Debita-V3-Contracts/contracts/buyOrders/buyOrder.sol#L99)the NFT


### Impact

The buyOrder creator will lose the NFT

### PoC

Path:
test/fork/BuyOrders/BuyOrder.t.sol

```solidity
function testpoc() public{
        vm.startPrank(seller);
        receiptContract.approve(address(buyOrderContract), receiptID);
        uint balanceBeforeAero = AEROContract.balanceOf(seller);
        address owner = receiptContract.ownerOf(receiptID);
       
        console.log("receipt owner before sell",owner);


        buyOrderContract.sellNFT(receiptID);
        address owner1 = receiptContract.ownerOf(receiptID);
        console.log("receipt owner after sell",owner1);
        //owner = buyOrderContract
        assertEq(owner1,address(buyOrderContract));
        
        
        vm.stopPrank();

    }

```
[PASS] testpoc() (gas: 242138)
Logs:
  receipt owner before sell 0x81B2c95353d69580875a7aFF5E8f018F1761b7D1
  receipt owner after sell 0xffD4505B3452Dc22f8473616d50503bA9E1710Ac
### Mitigation

After the buyOrder is completed,the NFT should be transferred  to the order creator

---
### Example 2

**Auto Label:** Missing or flawed ownership validation and access control leads to unchecked, irreversible authority, enabling centralized control and permanent system failure.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-04-teller-finance-judging/issues/35 

## Found by 
0x73696d616f, 0xAnmol, Afriaudit, AuditorPraise, EgisSecurity, MohammedRizwan
## Summary

`__Ownable_init()` is not called in `LenderCommitmentGroup_Smart::initialize()`, which will make the contract not have any owner.

## Vulnerability Detail

`LenderCommitmentGroup_Smart::initialize()` does not call `__Ownable_init()` and will be left without owner.  

## Impact

Inability to [pause](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L793) and [unpause](https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L800) borrowing in `LenderCommitmentGroup_Smart` due to having no owner, as these functions are `onlyOwner`.

## Code Snippet

https://github.com/sherlock-audit/2024-04-teller-finance/blob/main/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L158

## Tool used

Manual Review

Vscode

## Recommendation

Modify `LenderCommitmentGroup_Smart::initialize()` to call  `__Ownable_init()`:
```solidity
function initialize(
    ...
) external initializer returns (address poolSharesToken_) {
    __Ownable_init();
}
```



## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/teller-protocol/teller-protocol-v2-audit-2024/pull/13

---
### Example 3

**Auto Label:** Insufficient ownership and authorization checks enable unauthorized asset manipulation, leading to loss of control, unauthorized transfers, and potential theft through improper validation of caller identity or transaction order.  

**Original Text Preview:**

**Severity**: Critical

**Status**:  Resolved

**Description**

Since the CryptoPunks NFT collection not implementing the ERC721 standard, depositing of CryptoPunks NFTs is handled via a separate flow:
Token owner needs to call `offerPunkForSaleToAddress` and set the `toAddress` value to the address of the pool the token will be deposited into.
Token owner then calls the mint function of the PNFTToken contract.
The PNFTToken contract buys CryptoPunk from its owner. 

However, the mint function can be called by anyone, so the attacker can frontrun the legitimate mint call. The contract will buy the NFT and it will be on the caller's account even if the caller is not the owner of the token.
```js
 it("CryptoPunk can be stolen via mint frontrunning", async () => {
   // user 1 want to supply CryptoPunk NFT
   await app.cryptopunks
     .connect(app.user1)
     .offerPunkForSaleToAddress(tokenId, 0, app.pcryptopunks.address);


   //user 2 frontrun the mint call
   await app.pcryptopunks.connect(app.user2).mint(tokenId);
   await app.unitroller
     .connect(app.user2)
     .enterNFTMarkets([app.pcryptopunks.address]);


   expect(await app.pcryptopunks.ownerOf(tokenId)).to.equal(app.user2.address);
 });
```

**Recommendation**: 

It is advisable to verify that `msg.sender` is the rightful owner of the token, before buying a CryptoPunk.

---
