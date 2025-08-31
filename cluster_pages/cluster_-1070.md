# Cluster -1070

**Rank:** #51  
**Count:** 351  

## Label
Flawed ownership and approval validation leads to unauthorized access, denial of service, and incorrect token state, enabling users to bypass intended access controls or manipulate token balances.

## Cluster Information
- **Total Findings:** 351

## Examples

### Example 1

**Auto Label:** Failure to invalidate or reset approvals during state changes, allowing malicious re-approval or execution of outdated transactions.  

**Original Text Preview:**

The `DistributedRewardDistribution` allows distributors to commit a reward distribution and then have other distributors approve it. When the number of approvals hit a threshold, the commit is considered approved and the rewards are distributed out.

During the commit process, the current commit is stored in a mapping as shown below:

```solidity
commitments[fromBlock][toBlock] = commitment;
approves[fromBlock][toBlock] = 1;
```

The current distributor can always call this function, and if they use the same `fromBlock` and `toBlock`, they can always overwrite the previous commitment. This also resets the approves count to 1.

The issue is that in the `approve` function, a certain commit can only be approved once, no matter if its an overwritten or a new commit.

```solidity
require(!alreadyApproved[commitment][msg.sender], "Already approved");
    approves[fromBlock][toBlock]++;
    alreadyApproved[commitment][msg.sender] = true;
```

Now consider the following scenario:

1. There are 10 distributors. The limit for approvals is 5.
2. Distributor 1 adds a commitment for blocks 1-100. 3 other distributors approve it.
3. After 256 blocks, distributor 2 gains control. They call `commit` again and overwrite the previous commitment with a different one, but for the same 1-100 blocks.
4. After another 256 blocks, distributor 3 gains control and resubmits the original commitment made by distributor 1 for 1-100 blocks. Now, the distributors who voted before cannot vote anymore, since `alreadyApproved[commitment][msg.sender]` is set to true. So even though the contract labels it as approved by the distributor, the `approves` mapping does not reflect that.

This can deny votes in case of resubmissions, and also breaks the invariant that the `approves` should contain the same number as the number of unique addresses whose `alreadyApproved` mapping is true.

Since we are resetting the `approves` count to 1, we should use a fresh `alreadyApproved` mapping. Consider adding a `commitmentId` variable which increases on every commit. The approvals mapping should then implement an extra key and use `alreadyApproved[commitment][commitmentId][msg.sender]` instead, which will reset the already given approvals for the resubmitted commitment.

---
### Example 2

**Auto Label:** Flawed ownership and approval validation leads to unauthorized access, denial of service, and incorrect token state, enabling users to bypass intended access controls or manipulate token balances.  

**Original Text Preview:**

The `ownerOf` function in the `VotingEscrow` contract does not revert when queried with a non-existent token ID. This can lead to misleading behavior in the contract.

```solidity
function ownerOf(uint _tokenId) public view returns (address) {
    return idToOwner[_tokenId];
}
```

This is inconsistent with EIP-721:

```solidity
    /// @notice Find the owner of an NFT
    /// @dev NFTs assigned to zero address are considered invalid, and queries
    ///  about them do throw.
    /// @param _tokenId The identifier for an NFT
    /// @return The address of the owner of the NFT
    function ownerOf(uint256 _tokenId) external view returns (address);
```

---
### Example 3

**Auto Label:** Flawed ownership and approval validation leads to unauthorized access, denial of service, and incorrect token state, enabling users to bypass intended access controls or manipulate token balances.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

Inside `VotingEscrow.withdraw`, there is `_isApprovedOrOwner` check, which allows an approved `tokenId` spender to trigger this operation.

```solidity
    function withdraw(uint _tokenId) external nonreentrant {
>>>     assert(_isApprovedOrOwner(msg.sender, _tokenId));
        require(attachments[_tokenId] == 0 && !voted[_tokenId], "attached");

        LockedBalance memory _locked = locked[_tokenId];
        require(block.timestamp >= _locked.end, "The lock didn't expire");
        uint value = uint(int256(_locked.amount));

        locked[_tokenId] = LockedBalance(0, 0);
        uint supply_before = supply;
        supply = supply_before - value;

        // old_locked can have either expired <= timestamp or zero end
        // _locked has only 0 end
        // Both can have >= 0 amount
        _checkpoint(_tokenId, _locked, LockedBalance(0, 0));

        assert(IERC20(token).transfer(msg.sender, value));

        // Burn the NFT
        _burn(_tokenId);

        emit Withdraw(msg.sender, _tokenId, value, block.timestamp);
        emit Supply(supply_before, supply_before - value);
    }
```

```solidity
    function _isApprovedOrOwner(
        address _spender,
        uint _tokenId
    ) internal view returns (bool) {
        address owner = idToOwner[_tokenId];
        bool spenderIsOwner = owner == _spender;
>>>     bool spenderIsApproved = _spender == idToApprovals[_tokenId];
        bool spenderIsApprovedForAll = (ownerToOperators[owner])[_spender];
        return spenderIsOwner || spenderIsApproved || spenderIsApprovedForAll;
    }
```

However, inside `_burn`, when `approve` is called to reset `tokenId` approval, there is a check that prevent the approved spender to perform the operation.

```solidity
    function approve(address _approved, uint _tokenId) public {
        address owner = idToOwner[_tokenId];
        // Throws if `_tokenId` is not a valid NFT
        require(owner != address(0));
        // Throws if `_approved` is the current owner
        require(_approved != owner);
        // Check requirements
        bool senderIsOwner = (idToOwner[_tokenId] == msg.sender);
        bool senderIsApprovedForAll = (ownerToOperators[owner])[msg.sender];
>>>     require(senderIsOwner || senderIsApprovedForAll);
        // Set the approval
        idToApprovals[_tokenId] = _approved;
        emit Approval(owner, _approved, _tokenId);
    }
```

## Recommendations

Consider resetting `idToApprovals` directly inside the `_burn` function instead of calling `approve`.

---
