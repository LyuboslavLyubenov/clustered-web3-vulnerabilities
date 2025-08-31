# Cluster -1158

**Rank:** #97  
**Count:** 171  

## Label
Failure to validate or handle zero addresses leads to incorrect state updates, failed transactions, unintended fund transfers, or bypassed fee mechanisms—enabling loss of funds, reward inflation, or unauthorized control.

## Cluster Information
- **Total Findings:** 171

## Examples

### Example 1

**Auto Label:** Missing zero-amount validation in transfers leads to reverts on tokens that reject zero-value transfers, causing denial-of-service, failed operations, or unintended state changes.  

**Original Text Preview:**

**Description:** Per the ERC-4626 specification, the preview functions "MUST NOT revert due to vault specific user/global limits. MAY revert due to other conditions that would also cause mint/deposit/redeem/withdraw to revert".

```solidity
    function totalAccruedUSDe() public view returns (uint256) {
@>      uint pUSDeAssets = super.totalAssets();  // @audit - should return early if pUSDeAssets is zero to avoid reverting in the call below

@>      uint USDeAssets = _convertAssetsToUSDe(pUSDeAssets, true);
        return USDeAssets;
    }

    function _convertAssetsToUSDe (uint pUSDeAssets, bool withYield) internal view returns (uint256) {
@>      uint sUSDeAssets = pUSDeVault.previewRedeem(withYield ? address(this) : address(0), pUSDeAssets); // @audit - this can revert if passing yUSDe as the caller when it has no pUSDe balance
        uint USDeAssets = sUSDe.previewRedeem(sUSDeAssets);
        return USDeAssets;
    }

    function previewDeposit(uint256 pUSDeAssets) public view override returns (uint256) {
        uint underlyingUSDe = _convertAssetsToUSDe(pUSDeAssets, false);

@>      uint yUSDeShares = _valueMulDiv(underlyingUSDe, totalAssets(), totalAccruedUSDe(), Math.Rounding.Floor); // @audit - should explicitly handle the case where totalAccruedUSDe() returns zero rather than relying on _valueMulDiv() behaviour
        return yUSDeShares;
    }

    function previewMint(uint256 yUSDeShares) public view override returns (uint256) {
@>      uint underlyingUSDe = _valueMulDiv(yUSDeShares, totalAccruedUSDe(), totalAssets(), Math.Rounding.Ceil); // @audit - should explicitly handle the case where totalAccruedUSDe() and/or totalAssets() returns zero rather than relying on _valueMulDiv() behaviour
        uint pUSDeAssets = pUSDeVault.previewDeposit(underlyingUSDe);
        return pUSDeAssets;
    }

    function _valueMulDiv(uint256 value, uint256 mulValue, uint256 divValue, Math.Rounding rounding) internal view virtual returns (uint256) {
        return value.mulDiv(mulValue + 1, divValue + 1, rounding);
    }
```

As noted using `// @audit` tags in the code snippets above, `yUSDeVault::previewMint` and `yUSDeVault::previewDeposit` can revert for multiple reasons, including:
* when the pUSDe balance of the yUSDe vault is zero.
* when `pUSDeVault::previewRedeem` reverts due to division by zero in `pUSDeVault::previewYield`, invoked from `_convertAssetsToUSDe()` within `totalAccruedUSDe()`.

```solidity
     function previewYield(address caller, uint256 shares) public view virtual returns (uint256) {
        if (PreDepositPhase.YieldPhase == currentPhase && caller == address(yUSDe)) {

            uint total_sUSDe = sUSDe.balanceOf(address(this));
            uint total_USDe = sUSDe.previewRedeem(total_sUSDe);

            uint total_yield_USDe = total_USDe - Math.min(total_USDe, depositedBase);

@>          uint y_pUSDeShares = balanceOf(caller); // @audit - should return early if this is zero to avoid reverting below
@>          uint caller_yield_USDe = total_yield_USDe.mulDiv(shares, y_pUSDeShares, Math.Rounding.Floor);

            return caller_yield_USDe;
        }
        return 0;
    }

    function previewRedeem(address caller, uint256 shares) public view virtual returns (uint256) {
        return previewRedeem(shares) + previewYield(caller, shares);
    }
```

While a subset of these reverts could be considered "due to other conditions that would also cause deposit to revert", such as due to overflow, it would be better to explicitly handle these other edge cases. Additionally, even when called in isolation `yUSDeVault::totalAccruedUSDe` will revert if the pUSDe balance of the yUSDeVault is zero. Instead, this should simply return zero.

**Strata:** Fixed in commit [0f366e1](https://github.com/Strata-Money/contracts/commit/0f366e192941c875b651ee4db89b9fd3242a5ac0).

**Cyfrin:** Verified. The zero assets/shares edge cases are now explicitly handled in `yUSDeVault::_convertAssetsToUSDe` and pUSDeVault::previewYield`, including when the `yUSDe` state is not initialized as so will be equal to the zero address.

\clearpage

---
### Example 2

**Auto Label:** Failure to validate zero fee amounts leads to reverts during token transfers, causing denial-of-service or incorrect state updates in critical functions.  

**Original Text Preview:**

##### Description
The `P2pSuperformProxy` contract enforces:
```solidity
require(
    req.superformData.receiverAddress == address(this),
    P2pSuperformProxy__ReceiverAddressShouldBeP2pSuperformProxy(
        req.superformData.receiverAddress
            )
        );
```

  1.  Superform may enter an emergency state during which the normal withdrawal flow is interrupted.
  2.  In such an emergency, calling withdraw may only enqueue an emergency withdrawal request without actually transferring assets, so the proxy’s balance may remain unchanged. If withdraw reverts when the balance delta is zero, it will break the emergency flow and block further recovery.
  3.  During this emergency flow, tokens can be sent by the Superform administrator to **receiverAddress** (the proxy itself) — a path that never occurs in normal operation — and without a rescue mechanism these assets will become permanently locked.

##### Recommendation
We recommend:
  1.  Ensure that withdraw never reverts when the proxy’s balance change is zero. Treat a zero-delta result as a successful no-op to preserve the emergency withdrawal queue flow.
  2.  Implement a client-accessible rescue mechanism to recover any ERC-20 or native tokens held by the proxy as a result of the emergency flow.

> **Client's Commentary:**
> Fixed in https://github.com/p2p-org/p2p-lending-proxy/commit/b7b2a4ff5b321afa7d9edaddf62953411eab8ff0

---

---
### Example 3

**Auto Label:** Failure to validate ownership or recipient contracts enables unauthorized transfers, double-spending, or exploitation of reentrancy and custom receive logic.  

**Original Text Preview:**

**Description:** The purpose of the `IStory` interface is to allow 3 different entities (Admin, Creator and Collectors) to add "Stories" about a given artwork (NFT) which [describes the provenance of the artwork](https://docs.transientlabs.xyz/tl-creator-contracts/common-features/story-inscriptions). In the art world the "provenance" of an item can affect its status and price, so the `IStory` interface aims to facilitate an on-chain record of an artwork's "provenance".

`IStory` is designed to work like this:
* Creator/Admin can add a `CollectionsStory` for when a collection is added to a contract
* Creator of an artwork can add a `CreatorStory`
* Collector of an artwork can add one or more `Story` about their experience while holding the artwork

The `IStory` interface specification requires that `addCreatorStory` is only called by the creator:
```solidity
/// @notice Function to let the creator add a story to any token they have created
/// @dev This function MUST implement logic to restrict access to only the creator
function addCreatorStory(uint256 tokenId, string calldata creatorName, string calldata story) external;
```

But in the CryptoArt implementation of the `IStory` interface, the current token owner can always emit `CreatorStory` events:
```solidity
function addCreatorStory(uint256 tokenId, string calldata, /*creatorName*/ string calldata story)
    external
    onlyTokenOwner(tokenId)
{
    emit CreatorStory(tokenId, msg.sender, msg.sender.toHexString(), story);
}
```

**Impact:** As an NFT is sold or transferred to new owners, each subsequent owner can continue to add new `CreatorStory` events even though they aren't the Creator of the artwork. This corrupts the provenance of the artwork by allowing Collectors to add to the `CreatorStory` as if they were the Creator.

**Recommended Mitigation:** Only the Creator of an artwork should be able to emit the `CreatorStory` event. Currently the on-chain protocol does not record the address of the creator; this could either be added or `onlyOwner` could be used where the contract owner acts as a proxy for the creator.

**CryptoArt:**
Fixed in commit [94bfc1b](https://github.com/cryptoartcom/cryptoart-smart-contracts/commit/94bfc1b1454e783ef1fb9627cfaf0328ebe17b47#diff-1c61f2d0e364fa26a4245d1033cdf73f09117fbee360a672a3cb98bc0eef02adL439-R439).

**Cyfrin:** Verified.

\clearpage

---
