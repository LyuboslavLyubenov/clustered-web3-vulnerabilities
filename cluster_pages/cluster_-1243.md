# Cluster -1243

**Rank:** #399  
**Count:** 12  

## Label
Lack of proper state synchronization and access control leads to inconsistent token relationships, unauthorized cross-chain operations, and incorrect denomination handling, enabling misuse and system instability.

## Cluster Information
- **Total Findings:** 12

## Examples

### Example 1

**Auto Label:** Inconsistent access control and improper authorization lead to unauthorized cross-chain flows, enabling attackers to manipulate or redirect tokens and NFTs through flawed whitelisting and state management.  

**Original Text Preview:**

## Vulnerability Details
When Bridging NFTs between `L1<->L2`, we are whitelisting the NFT collection in the destination chain if it is not whitelisted, and since we are checking that depositing NFTs from the source chain should be from a whitelisted NFT, it is important to integrability between L1 and L2.

The problem is that we are only whitelisting the collection on the destination chain if it doesn't have an attachment to it on L1, whereas after deploying it we are whitelisting it. But if the collection already existed we are not whitelisting it if needed.

[Bridge.sol#L183-L196](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/ethereum/src/Bridge.sol#L183-L196)
```solidity
        if (collectionL1 == address(0x0)) {
            if (ctype == CollectionType.ERC721) {
                collectionL1 = _deployERC721Bridgeable( ... );
                // update whitelist if needed
❌️              _whiteListCollection(collectionL1, true);
            } else {
                revert NotSupportedYetError();
            }
        }
```

As we can see, whitelisting occurs only if the NFT collection does not exist on `L1`, so whitelisting the collection will not occur if the collection already exists.

We will discuss the scenario where this can occur and introduce problems in the `Proof of Concept` section.

## Proof of Concept

- whitelisting is disabled, and any NFT can be Bridged.
- NFTs are Bridged from L1, To L2.
- NFT collections on L1 are not whitelisted collections now (whitelisting is disabled).
- On L2 we are deploying addresses that maps to Collections on L1 and whitelist them. 
- the protocol enables whiteListing.
- NFTs collections that were created on L2 are now whitelisted and can be Bridged to L1, but still, these Collection L1 addresses are not whitelisted.
- We can Bridge from L2 to L1 easily as these collections are whitelisted.
- We can withdraw the NFT from L1 easily (there is no check for whitelisting when withdrawing).
- since `L1<->L2` mapping is already set. NFTs Collections are still `not-whitelisted` on L1.
- The process will end up with the ability to Bridge NFTs from `L2 to L1` as they are whitelisted on L2, but the inability to Bridge them from `L1 to L2` as they are not whitelisted on `L1`.

## Tools Used
Manual Review

## Recommendations
Whitelist the collection if it is newly deployed or if it already exists and is not whitelisted.

```diff
diff --git a/apps/blockchain/ethereum/src/Bridge.sol b/apps/blockchain/ethereum/src/Bridge.sol
index e62c7ce..e069544 100644
--- a/apps/blockchain/ethereum/src/Bridge.sol
+++ b/apps/blockchain/ethereum/src/Bridge.sol
@@ -188,13 +188,14 @@ contract Starklane is IStarklaneEvent, UUPSOwnableProxied, StarklaneState, Stark
                     req.collectionL2,
                     req.hash
                 );
-                // update whitelist if needed
-                _whiteListCollection(collectionL1, true);
             } else {
                 revert NotSupportedYetError();
             }
         }
 
+        // update whitelist if needed
+        _whiteListCollection(collectionL1, true);
+
         for (uint256 i = 0; i < req.tokenIds.length; i++) {
             uint256 id = req.tokenIds[i];
```

## Existence of the issue on the Starknet Side
This issue explains the flow from Ethereum to Starknet. the problem also existed in L2 side, where we are only whitelisting if there is no address for that collection on `L2`. If the `collection_l2 == 0`, we are deploying the collection on L2 and whitelist it. But if it is already existed we are just returning the address without whitelisting it if existed, same as that in L1.

[bridge.cairo#L440-L442](https://github.com/Cyfrin/2024-07-ark-project/blob/main/apps/blockchain/starknet/src/bridge.cairo#L440-L442)
```cairo
    fn ensure_erc721_deployment(ref self: ContractState, req: @Request) -> ContractAddress {
        ...
        let collection_l2 = verify_collection_address( ... );

        if !collection_l2.is_zero() {
❌️          return collection_l2;
        }
        ...
        // update whitelist if needed
        let (already_white_listed, _) = self.white_listed_list.read(l2_addr_from_deploy);
        if already_white_listed != true {
            _white_list_collection(ref self, l2_addr_from_deploy, true);
            ...
        }
        l2_addr_from_deploy
    }
```

So if we changed the order, the Collections can be whitelisted on `L1`, but not on `L2` (if the Proof of Concept was in the reverse order), Bridging these NFTs collections from L1 to L2 will be allowed, but we will not be able to Bridge them from `L2` to `L1`, as they are not whitelisted on `L2`.

**Recommendations**: the same as that in `L1`, we need to whitelist collections if they already exist.

```diff
diff --git a/apps/blockchain/starknet/src/bridge.cairo b/apps/blockchain/starknet/src/bridge.cairo
index 23cbf8a..1d3521e 100644
--- a/apps/blockchain/starknet/src/bridge.cairo
+++ b/apps/blockchain/starknet/src/bridge.cairo
@@ -438,6 +438,15 @@ mod bridge {
         );
 
         if !collection_l2.is_zero() {
+            // update whitelist if needed
+            let (already_white_listed, _) = self.white_listed_list.read(l2_addr_from_deploy);
+            if already_white_listed != true {
+                _white_list_collection(ref self, l2_addr_from_deploy, true);
+                self.emit(CollectionWhiteListUpdated {
+                    collection: l2_addr_from_deploy,
+                    enabled: true,
+                });
+            }
             return collection_l2;
         }
```

_NOTE: if the issue will be judged as two separate issues. Please, consider making this report as duplicate of both of them._

---
### Example 2

**Auto Label:** Inconsistent access control and improper authorization lead to unauthorized cross-chain flows, enabling attackers to manipulate or redirect tokens and NFTs through flawed whitelisting and state management.  

**Original Text Preview:**

**Severity**

**Impact**: High, malicious attacker can set L2 custom address to different address to break the bridge token.

**Likelihood**: Medium, attacker can front-ran the `registerTokenOnL2` to break the bridge token.

**Description**

tSQD is designed so that it can be bridged from Ethereum (L1) to Arbitrum (L2) via Arbitrum’s generic-custom gateway.
However, the `registerTokenOnL2` function, which sets the L2 token address via `gateway.registerTokenToL2`, is not currently restricted.

```solidity
  function registerTokenOnL2(
    address l2CustomTokenAddress,
    uint256 maxSubmissionCostForCustomGateway,
    uint256 maxSubmissionCostForRouter,
    uint256 maxGasForCustomGateway,
    uint256 maxGasForRouter,
    uint256 gasPriceBid,
    uint256 valueForGateway,
    uint256 valueForRouter,
    address creditBackAddress
  ) public payable {
    require(!shouldRegisterGateway, "ALREADY_REGISTERED");
    shouldRegisterGateway = true;

    gateway.registerTokenToL2{value: valueForGateway}(
      l2CustomTokenAddress, maxGasForCustomGateway, gasPriceBid, maxSubmissionCostForCustomGateway, creditBackAddress
    );

    router.setGateway{value: valueForRouter}(
      address(gateway), maxGasForRouter, gasPriceBid, maxSubmissionCostForRouter, creditBackAddress
    );

    shouldRegisterGateway = false;
  }
```

An attacker can front-run the `registerTokenOnL2` and put an incorrect address for `l2CustomTokenAddress` to break the bridge token. Once it is called, the L2 token cannot be changed inside the gateway.

**Recommendations**

Use the Ownable functionality inside tSQD and restrict `registerTokenOnL2` so that it can only be called by the owner/admin, as suggested by the Arbitrum bridge token design.

---
### Example 3

**Auto Label:** Lack of proper state synchronization and access control leads to inconsistent token relationships, unauthorized cross-chain operations, and incorrect denomination handling, enabling misuse and system instability.  

**Original Text Preview:**

## Severity: Low Risk

## Context:
- AstariaRouter.sol#L402-L409
- CollateralToken.sol#L330-L332
- LienToken.sol#L87-L90

## Description:
The main contracts `AstariaRouter`, `CollateralToken`, and `LienToken` all need to be aware of each other and form a connected triangle. They are all part of a single unit and perhaps are separated into three different contracts due to code size and needing to have two individual ERC721 tokens. Their authorised filing structure is as follows:
- Note that one cannot file for `CollateralToken` to change `LienToken` as the value of `LienToken` is only set during the `CollateralToken`'s initialisation.

If one files to change one of these nodes and forgets to check or update the links between these contracts, the triangle above would be broken.

## Recommendation:
To ensure the connectivity of the above triangle:
1. Each contract/node has two storage variables for the other two nodes.
2. Each node should have an authorised endpoint to file updates for the other two nodes.
3. Once a link has been established between two nodes, the changes should be propagated to the third node to ensure connectivity.

One can also have a different design where there is an external contract that manages the nodes and their links:

To swap one of these nodes, we would:
1. Each contract/node has two storage variables for the other two nodes.
2. Each node has a restricted `fileNode` endpoint which accepts one or two arguments (depends on the design) for the changed/swapped nodes, and only the `NodeManager` can call this to update the internal storage of the called node pointing to the other nodes.
3. `NodeManager` should have an authorised endpoint that can be called to swap one or more of the nodes, which the `NodeManager` would need to propagate the changes to all the nodes. The new node would need to set its `NodeManager` upon initialisation or construction.

If the above changes are not applied, we need to monitor that the triangle is intact when a node is swapped.

---
