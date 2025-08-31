# Cluster -1242

**Rank:** #304  
**Count:** 24  

## Label
Lack of proper state synchronization and access control leads to inconsistent token relationships, unauthorized cross-chain operations, and incorrect denomination handling, enabling misuse and system instability.

## Cluster Information
- **Total Findings:** 24

## Examples

### Example 1

**Auto Label:** **Insufficient state validation in cross-chain token bridging leads to incorrect token mapping, double-bridging, and unauthorized minting or fund siphoning.**  

**Original Text Preview:**

In order to bridge ERC-20 tokens to L2s, the `ZkStack_Adapter` and `ZkStack_CustomGasToken_Adapter` contracts first [approve](https://github.com/across-protocol/contracts/blob/5a0c67c984d19a3bb843a4cec9bb081734583dd1/contracts/chain-adapters/ZkStack_Adapter.sol#L147) the relevant amount of tokens to the `SHARED_BRIDGE` and then [invoke the `requestL2TransactionTwoBridges` function](https://github.com/across-protocol/contracts/blob/5a0c67c984d19a3bb843a4cec9bb081734583dd1/contracts/chain-adapters/ZkStack_Adapter.sol#L148-L160) of the `Bridgehub` contract, where they specify the second bridge to be used as `BRIDGE_HUB.sharedBridge()`. The `requestL2TransactionTwoBridges` function then [calls the `bridgehubDeposit` function](https://etherscan.io/address/0x509da1be24432f8804c4a9ff4a3c3f80284cdd13#code#F1#L290) on the second bridge, which [transfers tokens from the contract which initially called the Bridge Hub](https://github.com/matter-labs/era-contracts/blob/aafee035db892689df3f7afe4b89fd6467a39313/l1-contracts/contracts/bridge/L1SharedBridge.sol#L295).

However, the token approval given by the adapters is always for the immutable `SHARED_BRIDGE` address, yet the `BRIDGE_HUB.sharedBridge()` address, specified as the second bridge, returns the current value of the [`sharedBridge` variable](https://etherscan.io/address/0x509da1be24432f8804c4a9ff4a3c3f80284cdd13#code#F1#L19). Although it is unlikely that the variable will change in the future, [it is nonetheless possible](https://etherscan.io/address/0x509da1be24432f8804c4a9ff4a3c3f80284cdd13#code#F1#L111), and if that happens, none of the ZkStack adapters will be able to bridge tokens as the allowance will be given to the previous `sharedBridge` address.

Consider removing the `SHARED_BRIDGE` variable and always accessing the `sharedBridge` variable through `BRIDGE_HUB.sharedBridge()`.

***Update:** Acknowledged, not resolved. The decision has been made to redeploy the adapters in case when the `sharedBridge` variable changes and not to call `BRIDGE_HUB.sharedBridge()` in order to save gas. `BRIDGE_HUB.sharedBridge()` calls have been replaced with the `SHARED_BRIDGE` variable accesses in the commit [3d260d7](https://github.com/across-protocol/contracts/pull/748/commits/3d260d73ba7aa3a63bdf83aad4d9ab6864cb27fc) in order to reduce gas cost further. The team stated:*

> *This is true, but we think this may be one of the few where we may want to have this behavior. This is because we call these adapters often, and since they are deployed on L1, they can become fairly expensive. For this reason, it is particularly important to minimize the gas cost whenever possible, and this is one such shortcut we take. [...] Particularly, in the event the bridge \_does* change, the adapter calls would just revert, and we would need to redeploy. To be clear, if the shared bridge does change, we will need to deploy a new adapter. The hope is that in the long run, the cost of redeploying will be cheaper than making an extra call on the adapter for each new transaction.\_

---
### Example 2

**Auto Label:** **Improper state validation and silent failure in token transfers lead to fund loss, inconsistent balances, and potential reentrancy-like attacks.**  

**Original Text Preview:**

Severity: Low Risk
Context: WrappedVault.sol#L534-L537
Description: The redeem function has an incorrect sequence of asset transfers that causes it to always
revert, making it impossible for users to execute the function successfully.
Here's the breakdown of the issue:
• L534: assets = VAULT.redeem(shares, receiver, address(this)); this will burn the amount of
shares of the underlying vault in the contract and transfer underlying asset to receiver .
• L536: _burn(owner, shares); this will burn the shares of the WrappedVault .
• L537: DEPOSIT_ASSET.safeTransfer(receiver, assets); this line will revert bacause there is no
DEPOSIT_ASSET balance in this contract, and DEPOSIT_ASSET was already transfered in previous line.
Recommendation: Modify the redeem function to ensure assets are transferred to the receiver only
once per operation, remove redundant line L537: DEPOSIT_ASSET.safeTransfer(receiver, assets); .
Royco: Fixed in commit 1e952741.
Spearbit: Fix conﬁrmed.

---
### Example 3

**Auto Label:** **Insufficient state validation in cross-chain token bridging leads to incorrect token mapping, double-bridging, and unauthorized minting or fund siphoning.**  

**Original Text Preview:**

**Severity**: Medium	

**Status**: Resolved

**Description**

The `registerToken()` function within the `WrappedTokenBridge.sol` smart contract is used to register a certain pair of tokens for bridging them:

localToRemote[localToken][remoteChainId] = remoteToken;        remoteToLocal[remoteToken][remoteChainId] = localToken;

These mappings are used by bridge functions in order to check if certain token is allowed to be bridged:

```solidity
address remoteToken = localToRemote[localToken][remoteChainId];
       if (remoteToken == address(0)) {
           revert UnsupportedToken();
       }
```

However, there is no possibility for unregistering certain token or certain pair is no longer wanted to be supported. This functionality is only available when bridging to BRC20 or bridging to Runes as they implement the `setCanBeBridgedToBrc20()` and `setCanBeBridgedToRunes` but not for the rest of bridge types.

It is possible that in a certain point of time a certain token or certain pair is no longer wanted to be supported for any reason, security for example.

The same issue is present within the `OriginalTokenBridge.sol` smart contract.

**Recommendation**:

Implement a function that allows the owner to unregister certain tokens.

---
