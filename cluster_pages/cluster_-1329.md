# Cluster -1329

**Rank:** #291  
**Count:** 26  

## Label
Insufficient ownership and authorization checks enable unauthorized asset manipulation, leading to loss of control, unauthorized transfers, and potential theft through improper validation of caller identity or transaction order.

## Cluster Information
- **Total Findings:** 26

## Examples

### Example 1

**Auto Label:** Failure to validate or restrict critical address or state updates leads to irreversible control loss or unauthorized contract takeover.  

**Original Text Preview:**

## Data Validation Report

## Difficulty: Low

## Type: Data Validation

### File Location
`nominators-v2/src/contract/nominators.fc`

### Description
When called, the `op::change_address` operation immediately sets the owner or controller address to a new value. The use of a single step to make such a critical change is error-prone; if the function is called with erroneous input, the results could be irrevocable, leading to permanent loss of important owner functionality.

```fc
() op_change_address(int value, slice in_msg) impure {
    ;; Parse parameters
    var id = in_msg~load_uint(8);
    var new_address = in_msg~load_msg_addr();
    in_msg.end_parse();
    
    ;; Throw if new_address is already owner or controller
    if (equal_slices(ctx_owner, new_address) | equal_slices(ctx_controller, new_address)) {
        throw(error::invalid_message());
    }
    
    ;; Update address
    if (id == 0) {
        ctx_owner = new_address;
    } elseif (id == 1) {
        ctx_controller = new_address;
    } else {
        throw(error::invalid_message());
    }
}
```
*Figure 2.1: The `op_change_address` function, callable by the contract owner (nominators-v2/src/contract/modules/op-owner.fc#L1–L20)*

### Recommendations
- **Short term:** Implement a two-step process for all irrevocable critical operations. The `treasure-v0` contract from the holders repository uses a two-step process for ownership transfer, which can be implemented in the nominators contract to resolve this issue.
  
- **Long term:** Identify and document all possible actions that can be taken by privileged accounts, along with their associated risks. This will facilitate reviews of the codebase and prevent future mistakes.

---
### Example 2

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
### Example 3

**Auto Label:** Lack of ownership verification enables unauthorized actors to perform privileged operations, bypassing access controls and leading to full or partial contract takeover.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-08-perennial-v2-update-3-judging/issues/52 

## Found by 
Nyx, bin2chen, eeyore, neko\_nyaa, oot2k, panprog, silver\_eth, volodya
## Summary

An attacker can set himself as an `extension`, which is an `allowed protocol-wide operator`. As such, he can act on an account's behalf in all its positions and, for example, withdraw its collateral.

## Vulnerability Detail

A new authorization functionality was introduced in Perennial 2.3 update to allow for signers and extensions to act on behalf of the account. Unfortunately, the `updateExtension()` function within the `MarketFactory` is missing the `onlyOwner` access control modifier.

```solidity
File: MarketFactory.sol
100:@>   function updateExtension(address extension, bool newEnabled) external {
101:         extensions[extension] = newEnabled;
102:         emit ExtensionUpdated(extension, newEnabled);
103:     }
```

This `extensions` mapping is later used in the `authorization()` function to determine if the sender is an account operator:

```solidity
File: MarketFactory.sol
77:     function authorization(
78:         address account,
79:         address sender,
80:         address signer,
81:         address orderReferrer
82:     ) external view returns (bool isOperator, bool isSigner, UFixed6 orderReferralFee) {
83:         return (
84:@>           account == sender || extensions[sender] || operators[account][sender],
85:             account == signer || signers[account][signer],
86:             referralFees(orderReferrer)
87:         );
88:     }
```

The `authorization()` function is used within the `Market` contract to authorize the order in the name of the account:

```solidity
File: Market.sol
500:         // load factory metadata
501:         (updateContext.operator, updateContext.signer, updateContext.orderReferralFee) =
502:@>           IMarketFactory(address(factory())).authorization(context.account, msg.sender, signer, orderReferrer);
503:         if (guaranteeReferrer != address(0)) updateContext.guaranteeReferralFee = guaranteeReferralFee;
504:     }
```
```solidity
File: InvariantLib.sol
78:         if (
79:             !updateContext.signer &&                                            // sender is relaying the account's signed intention
80:@>           !updateContext.operator &&                                          // sender is operator approved for account
81:             !(newOrder.isEmpty() && newOrder.collateral.gte(Fixed6Lib.ZERO))    // sender is depositing zero or more into account, without position change
82:         ) revert IMarket.MarketOperatorNotAllowedError();
```

As can be seen, anyone without authorization can set himself as an extension and act as the operator of any account, leading to the loss of all funds.

## Impact

- Loss of funds.
- Missing access control.

## Code Snippet

https://github.com/sherlock-audit/2024-08-perennial-v2-update-3/blob/main/perennial-v2/packages/perennial/contracts/MarketFactory.sol#L100-L103
https://github.com/sherlock-audit/2024-08-perennial-v2-update-3/blob/main/perennial-v2/packages/perennial/contracts/MarketFactory.sol#L77-L88
https://github.com/sherlock-audit/2024-08-perennial-v2-update-3/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L500-L504
https://github.com/sherlock-audit/2024-08-perennial-v2-update-3/blob/main/perennial-v2/packages/perennial/contracts/libs/InvariantLib.sol#L78-L82

## Tool used

Manual Review

## Recommendation

Add the `onlyOwner` modifier to the `MarketFactory.updateExtension()` function.



## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/equilibria-xyz/perennial-v2/pull/443

---
