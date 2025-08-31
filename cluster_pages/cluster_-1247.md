# Cluster -1247

**Rank:** #269  
**Count:** 31  

## Label
Insufficient ownership and authorization checks enable unauthorized asset manipulation, leading to loss of control, unauthorized transfers, and potential theft through improper validation of caller identity or transaction order.

## Cluster Information
- **Total Findings:** 31

## Examples

### Example 1

**Auto Label:** Inadequate authorization checks in NFT ownership and transfer logic enable unauthorized access, asset control, or fund theft through flawed validation of sender, receiver, or operator roles.  

**Original Text Preview:**

## Security Risk in Transaction Authorization

The current implementation allows the user to sign and submit transactions (`signAndSubmitTransaction`) without explicitly displaying or simulating transaction parameters. This lack of transparency creates a significant risk that a user might unknowingly authorize a malicious or fraudulent transaction, potentially compromising their account or funds.

## Code Snippet

```typescript
export const onRpcRequest: OnRpcRequestHandler = async ({
  origin,
  request,
}) => {
  [...]
  case 'signAndSubmitTransaction': {
    const account = await getAccount();
    const result = await snap.request({
      method: 'snap_dialog',
      params: {
        type: 'confirmation',
        content: (
          <Box>
            <Text>
              <Bold>{origin}</Bold> wants to sign a transaction
            </Text>
            <Text>
              using <Bold>{account.accountAddress.toString()}</Bold> account
            </Text>
            <Text>Confirm to sign this transaction.</Text>
          </Box>
        ),
      },
    });
    [...]
  }
  [...]
}
```

## Remediation

Display the transaction parameters before confirmation, or simulate the transaction’s execution and show a summary of its expected outcome to help users understand its effects.

## Patch

Fixed in `de5a944`. Nightly is also committing to introduce a full simulation.

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

**Auto Label:** Inadequate authorization checks in NFT ownership and transfer logic enable unauthorized access, asset control, or fund theft through flawed validation of sender, receiver, or operator roles.  

**Original Text Preview:**

The `accountId` owners can not invoke the `SessionOrderModule.executeSessionOrdersBySig` function and the `SessionOrderModule.executeSessionOrders` function while they do not initiate themselves via the `SessionPermissionModule._initiateOrUpdateSession` function though the `accountId` owners are authorized to trade by default.

```solidity
    function _executeSessionOrders(
        address signer,
        uint128 accountId,
        MatchOrderDetails[] memory orders
    )
        internal
        returns (bytes[] memory)
    {
>>      if (!SessionPermissions.isPermissionActive(accountId, signer)) {
            revert Errors.Unauthorized(signer);
        }

        return executeMatchOrders(accountId, orders, Configuration.getCoreProxyAddress());
    }
<...>

library SessionPermissions {
<...>
    function isPermissionActive(uint128 accountId, address target) internal view returns (bool) {
        uint256 activeUntil = load(accountId).permissions[target];

        return activeUntil >= block.timestamp;
    }
```

Consider checking if the `msg.sender` is the `accountId` owner in the `_executeSessionOrders` function:

```diff
+import { isOwner } from "../libraries/AccountPermissions.sol";
<...>
    function _executeSessionOrders(
        address signer,
        uint128 accountId,
        MatchOrderDetails[] memory orders
    )
        internal
        returns (bytes[] memory)
    {
-       if (!SessionPermissions.isPermissionActive(accountId, signer)) {
+       if (!SessionPermissions.isPermissionActive(accountId, signer) &&
+           !isOwner(accountId, msg.sender, Configuration.getCoreProxyAddress())) {
            revert Errors.Unauthorized(signer);
        }

        return executeMatchOrders(accountId, orders, Configuration.getCoreProxyAddress());
    }
```

---
