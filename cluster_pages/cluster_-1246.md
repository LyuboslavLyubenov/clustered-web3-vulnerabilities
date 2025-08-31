# Cluster -1246

**Rank:** #355  
**Count:** 17  

## Label
Failure to validate NFT ownership before critical operations, enabling spoofing, unauthorized acquisition, or exposure of invalid metadata due to insufficient checks against ERC721 ownership rules.

## Cluster Information
- **Total Findings:** 17

## Examples

### Example 1

**Auto Label:** Failure to properly invalidate or zero out NFT ownership or state upon burn, leading to persistent, invalid token references that enable unauthorized transfers, misuse, or stale metadata.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

VotingEscrow merge(), split(), and withdraw() functions burn the user's NFT without claiming pending rewards from the bribe system (via getReward). Since claiming bribe rewards requires ownership of the NFT (isApprovedOrOwner check), burning the NFT makes it impossible to access unclaimed rewards afterward.

In the merge() function:

```
locked[_from] = LockedBalance(0, 0);
_checkpoint(_from, _locked0, LockedBalance(0, 0));
_burn(_from);
```

The getReward() function that users must call to claim bribe rewards checks ownership:

`require(IVotingEscrow(_ve).isApprovedOrOwner(msg.sender, tokenId));
`
Burning the NFT removes this ownership, making it impossible to later call getReward.

## Recommendations

Reset token amounts instead of directly burning nft.

---
### Example 2

**Auto Label:** Failure to properly invalidate or zero out NFT ownership or state upon burn, leading to persistent, invalid token references that enable unauthorized transfers, misuse, or stale metadata.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

VotingEscrow merge(), split(), and withdraw() functions burn the user's NFT without claiming pending rewards from the bribe system (via getReward). Since claiming bribe rewards requires ownership of the NFT (isApprovedOrOwner check), burning the NFT makes it impossible to access unclaimed rewards afterward.

In the merge() function:
```
locked[_from] = LockedBalance(0, 0);
_checkpoint(_from, _locked0, LockedBalance(0, 0));
_burn(_from);
```
The getReward() function that users must call to claim bribe rewards checks ownership:

`require(IVotingEscrow(_ve).isApprovedOrOwner(msg.sender, tokenId));
`
Burning the NFT removes this ownership, making it impossible to later call getReward.


## Recommendations

Reset token amounts instead of directly burning nft.

---
### Example 3

**Auto Label:** Failure to properly invalidate or zero out NFT ownership or state upon burn, leading to persistent, invalid token references that enable unauthorized transfers, misuse, or stale metadata.  

**Original Text Preview:**

**Description:** When a user is added to the blacklist, the NFT which they already paid for is burned:
```solidity
function _addToBlacklist(address account) internal {
    // *snip: code not relevant *//

    if (balanceOf(account) > 0) {
        uint256 tokenId = tokenOfOwnerByIndex(account, 0); // Get first token
        _burn(tokenId); // Burn the token
    }
}
```

But when they are removed from the blacklist, they do not receive a free NFT to make up for their previously burned one, nor is there any flag set that would enable them to mind their NFT again but without paying a fee:
```solidity
function _removeFromBlacklist(address account) internal {
    if (!s_blacklist[account]) revert SoulBoundToken__NotBlacklisted(account);

    s_blacklist[account] = false;
    emit RemovedFromBlacklist(account);
}
```

**Impact:** A user who bought an NFT, then was blacklisted, then removed from the blacklist will have to pay twice to get the NFT.

**Recommended Mitigation:** This doesn't seem fair; if a user had an NFT burned when they were blacklisted, they should receive a free NFT back if later removed from the blacklist.

**Evo:**
Acknowledged; in the unlikely case a user is blacklisted due to admin error then subsequently removed from the blacklist, the DAO will compensate the user via a community vote.

---
