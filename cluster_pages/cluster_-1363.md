# Cluster -1363

**Rank:** #119  
**Count:** 135  

## Label
Failure to validate recipient contracts during ERC721/ERC1155 minting, leading to asset trapping or irreversible loss due to missing receiver interface checks.

## Cluster Information
- **Total Findings:** 135

## Examples

### Example 1

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
### Example 2

**Auto Label:** Failure to validate ownership or recipient contracts enables unauthorized transfers, double-spending, or exploitation of reentrancy and custom receive logic.  

**Original Text Preview:**

## Severity

Medium Risk

## Description

The `publicMint()` and `preMint()` in `NftFactory.sol` will call the corresponding functions in the NFT contract:

```solidity
function publicMint(address nft, uint256 mintAmount) external returns (uint256) {
    uint256 tokenId = LendingNFT(nft).publicMint(mintAmount);
    return tokenId;
}

function preMint(address nft, uint256 mintAmount, uint256 maxMintAmount, bytes memory signature)
    external
    returns (uint256)
{
    uint256 tokenId = LendingNFT(nft).preMint(mintAmount, maxMintAmount, signature);
    return tokenId;
}
```

In the `LendingNFT.sol` contract, both `publicMint()` and `preMint()` will perform transfers from `msg.sender` through `safeTransferFrom()`.

The issue here is that the `LendingNFT.sol` contract is not approved to transfer payment tokens before calling the minting function through `NftFactory()`, meaning that `safeTransferFrom()` cannot retrieve assets from the `NftFactory.sol` contract. Therefore, calling `NftFactory.publicMint()`/`preMint()` will revert.

## Location of Affected Code

File: [NftFactory.sol#L59-L76](https://github.com/tokyoweb3/HARVESTFLOW_Ver.2/blob/05f0814caa51dcb034dd01f610315d4c6efedce8/contracts/src/NftFactory.sol#L59-L76)

## Impact

The mint functions in `NFTFactory.sol` do not function properly.

## Recommendation

In the `publicMint()` and `preMint()` of `NftFactory.sol`, first, obtain assets from `msg.sender`, then approve them to the `LendingNFT.sol` contract, and finally call `LendingNFT.publicMint()`/`preMint()`.

## Team Response

Fixed.

---
### Example 3

**Auto Label:** Failure to validate ownership or recipient contracts enables unauthorized transfers, double-spending, or exploitation of reentrancy and custom receive logic.  

**Original Text Preview:**

## Diﬃculty: Low

## Type: Denial of Service

### Description
In the current implementation, when an EOA upgrades to a smart wallet, it will no longer be able to receive NFTs due to a lack of token callbacks in the current implementation. Per the ERC-721 and ERC-1155 standards, when an NFT is transferred using `safeTransfer`, the method checks if the receiving address is an EOA or a smart contract. For the transfer to succeed, the receiver must be an EOA or a smart contract that implements `IERC721Receiver.onERC721Received`. The EIP-7702 standard transforms an EOA into a smart contract, so `IERC721Receiver.onERC721Received` must be implemented in the smart wallet implementation; otherwise, the user will not be able to continue to receive NFTs.

### Exploit Scenario
A user attempts to purchase an NFT from a marketplace, but their transaction will always revert due to a lack of token callbacks in their smart wallet implementation.

### Recommendations
- **Short term**: Add support for ERC-721 and ERC-1155 in the delegate contract; add support directly, or make a receiver contract that the delegate contract will inherit from.
- **Long term**: Identify other standards the Otim protocol is planned to be compatible with and test how they behave with Otim smart wallets.

---
