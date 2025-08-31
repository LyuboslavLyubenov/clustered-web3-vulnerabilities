# Cluster -1112

**Rank:** #388  
**Count:** 13  

## Label
Redundant state mutations and unnecessary storage operations that waste gas, introduce inefficiency, and risk unintended state inconsistencies without affecting functionality.

## Cluster Information
- **Total Findings:** 13

## Examples

### Example 1

**Auto Label:** Redundant state mutations and unnecessary storage operations that waste gas, introduce inefficiency, and risk unintended state inconsistencies without affecting functionality.  

**Original Text Preview:**

**Description:** Both `pUSDeVault` and `yUSDeVault` inherit the `PreDepositVault` which in turn inherits the `PreDepositPhaser`; however, there is an inconsistency between the state of `pUSDe::currentPhase`, which is updated when the phase changes, and `yUSDe::currentPhase`, which is never updated and is thus always the default `PointsPhase` variant. This is assumedly not an issue given that this state is never needed for the yUSDe vault, though a view function is exposed by virtue of the state variable being public which could cause confusion.

**Recommended Mitigation:** The simplest solution would be modifying this state to be internal by default and only expose the corresponding view function within `pUSDeVault`.

**Strata:** Fixed in commit [aac3b61](https://github.com/Strata-Money/contracts/commit/aac3b617084fb5a06b29728a9f52e5884b062b6a).

**Cyfrin:** Verified. The `yUSDeVault` now returns the `pUSDeVault` phase state.

\clearpage

---
### Example 2

**Auto Label:** Redundant state mutations and unnecessary storage operations that waste gas, introduce inefficiency, and risk unintended state inconsistencies without affecting functionality.  

**Original Text Preview:**

#### Resolution

The Linea team has fixed the finding in [PR 354](https://github.com/Consensys/linea-monorepo/pull/354).


#### Description

In the function `_computePublicInput` of the `LineaRollup` contract, the parameter `_endBlockNumber` is redundant as it’s included in the parameter `_finalizationData`, the function can load the parameter from `_finalizationData` directly.

#### Examples

**contracts/contracts/LineaRollup.sol:L683-L689**

```
function _computePublicInput(
  FinalizationDataV3 calldata _finalizationData,
  bytes32 _lastFinalizedShnarf,
  bytes32 _finalShnarf,
  uint256 _lastFinalizedBlockNumber,
  uint256 _endBlockNumber
) private pure returns (uint256 publicInput) {

```

**contracts/contracts/interfaces/l1/ILineaRollup.sol:L102-L115**

```
struct FinalizationDataV3 {
  bytes32 parentStateRootHash;
  uint256 endBlockNumber;
  ShnarfData shnarfData;
  uint256 lastFinalizedTimestamp;
  uint256 finalTimestamp;
  bytes32 lastFinalizedL1RollingHash;
  bytes32 l1RollingHash;
  uint256 lastFinalizedL1RollingHashMessageNumber;
  uint256 l1RollingHashMessageNumber;
  uint256 l2MerkleTreesDepth;
  bytes32[] l2MerkleRoots;
  bytes l2MessagingBlocksOffsets;
}

```
#### Recommendation

Remove the parameter `_endBlockNumber` and load it from `_finalizationData` inside the function using `calldatacopy`

---
### Example 3

**Auto Label:** Redundant state or logic leads to unnecessary storage costs, confusion, and potential exploitation through inconsistent or misaligned interfaces.  

**Original Text Preview:**

##### Description

In the `VaultNotifier` contract, there is a function `notifyWethDeposit` that declares a parameter `wethAmount` which is never used within the function body.

```
function notifyWethDeposit(address vault, address addr, uint256 amount, uint256 wethAmount) public onlyVault {
    emit WETHDeposit(vault, addr, amount);
}
```

##### Score

Impact:   
Likelihood:

##### Recommendation

To address this issue, consider one of the following options:

* Remove the unused parameter:

```
function notifyWethDeposit(address vault, address addr, uint256 amount) public onlyVault {
    emit WETHDeposit(vault, addr, amount);
}
```

* If the `wethAmount` is intended to be used but was overlooked, update the function to properly utilize this parameter:

```
event WETHDeposit(address indexed vault, address indexed addr, uint256 amount, uint256 wethAmount);

function notifyWethDeposit(address vault, address addr, uint256 amount, uint256 wethAmount) public onlyVault {
    emit WETHDeposit(vault, addr, amount, wethAmount);
}
```

  

###

##### Remediation

**ACKNOWLEDGED :** The **AltcoinIst team** acknowledged the issue**.**

---
