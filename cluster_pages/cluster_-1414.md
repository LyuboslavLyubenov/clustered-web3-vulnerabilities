# Cluster -1414

**Rank:** #307  
**Count:** 24  

## Label
Inconsistent or incorrect total supply tracking due to unit errors, flawed state updates, or misuse of supply variables, leading to inaccurate supply reporting, division-by-zero, or invalid state transitions.

## Cluster Information
- **Total Findings:** 24

## Examples

### Example 1

**Auto Label:** Inconsistent or incorrect total supply tracking due to unit errors, flawed state updates, or misuse of supply variables, leading to inaccurate supply reporting, division-by-zero, or invalid state transitions.  

**Original Text Preview:**

The circulating amount of tokens on the current chain includes the bridged tokens.

```solidity
    function circulatingSupply() public view virtual override returns (uint) {
        return innerToken.totalSupply();
    }
```

Consider tracking a separate variable for bridged tokens to subtract it from `innerToken.totalSupply()`.

---
### Example 2

**Auto Label:** Inconsistent or incorrect total supply tracking due to unit errors, flawed state updates, or misuse of supply variables, leading to inaccurate supply reporting, division-by-zero, or invalid state transitions.  

**Original Text Preview:**

**Description:** When `MembershipERC1155::mint` is invoked during a call to `MembershipFactory::joinDAO`, the `totalSupply` increment is performed after the call to `ERC1155::_mint`:

```solidity
function mint(address to, uint256 tokenId, uint256 amount) external override onlyRole(OWP_FACTORY_ROLE) {
    _mint(to, tokenId, amount, "");
    totalSupply += amount * 2 ** (6 - tokenId); // Update total supply with weight
}
```

While there does not appear to be any immediate impact, this is in violation of the Checks-Effects-Interactions (CEI) pattern and thus potentially unsafe due to the [invocation](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/49c0e4370d0cc50ea6090709e3835a3091e33ee2/contracts/token/ERC1155/ERC1155.sol#L285) of [`ERC1155::_doSafeTransferAcceptanceCheck`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/49c0e4370d0cc50ea6090709e3835a3091e33ee2/contracts/token/ERC1155/ERC1155.sol#L467-L486):

```solidity
if (to.isContract()) {
    try IERC1155Receiver(to).onERC1155Received(operator, from, id, amount, data) returns (bytes4 response) {
        if (response != IERC1155Receiver.onERC1155Received.selector) {
            revert("ERC1155: ERC1155Receiver rejected tokens");
        }
```

**Impact:** There does not appear to be any immediate impact, although any code executed within a receiver smart contract will work with an incorrect `totalSupply` state.

**Recommended Mitigation:** Consider increasing the `totalSupply` before the call to `_mint()`.

**One World Project:** Updated the pattern in [`30465a3`](https://github.com/OneWpOrg/smart-contracts-blockchain-1wp/commit/30465a3197adea883413298a9ac17fe8a1f0289e).

**Cyfrin:** Verified. State changes now done before external call is made.

---
### Example 3

**Auto Label:** Inconsistent supply tracking due to manual state management and lack of synchronization, leading to logic errors, token collisions, and invalid state transitions.  

**Original Text Preview:**

## Vulnerability Overview

The vulnerability concerns the integrity of the snapshot process. A successful snapshot relies on accurately capturing the total supply of tokens and distinguishing between those that are counted (unlocked) and those that are not (locked). 

The `join` function allows two different tokens to be merged into one, potentially altering the balances and total supply of tokens mid-snapshot. If tokens that are part of the snapshot join with those that are not, `total_supply` will no longer be equal to `unlocked_sum + locked_sum`.

## Remediation

Employ a locking mechanism to block `shared_token::join` during the snapshot process.

## Patch

Resolved in commit `f6fa43b`.

---
