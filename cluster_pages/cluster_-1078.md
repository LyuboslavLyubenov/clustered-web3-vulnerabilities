# Cluster -1078

**Rank:** #52  
**Count:** 339  

## Label
Failure to validate transaction rekeying and state changes enables attackers to rekey accounts or double-spend funds, leading to unauthorized withdrawals and loss of asset value through insufficient transaction semantics and missing access controls.

## Cluster Information
- **Total Findings:** 339

## Examples

### Example 1

**Auto Label:** Inconsistent access control and inadequate authorization checks allow unauthorized users to manipulate locked balances, leading to unauthorized state changes and potential fund exposure.  

**Original Text Preview:**

`increase_amount` and `deposit_for` perform the same operation but have inconsistent restrictions: `increase_amount` only allows the approved spender or owner of the `tokenId` to execute the operation, whereas `deposit_for` can be called by anyone.

```solidity
    function deposit_for(uint _tokenId, uint _value) external nonreentrant {
        LockedBalance memory _locked = locked[_tokenId];
        require(_value > 0); // dev: need non-zero value
        require(_locked.amount > 0, "No existing lock found");
        require(
            _locked.end > block.timestamp,
            "Cannot add to expired lock. Withdraw"
        );
        _deposit_for(
            _tokenId,
            _value,
            0,
            _locked,
            DepositType.DEPOSIT_FOR_TYPE
        );
    }
```

```solidity
    function increase_amount(uint _tokenId, uint _value) external nonreentrant {
>>>     assert(_isApprovedOrOwner(msg.sender, _tokenId));

        LockedBalance memory _locked = locked[_tokenId];

        assert(_value > 0); // dev: need non-zero value
        require(_locked.amount > 0, "No existing lock found");
        require(
            _locked.end > block.timestamp,
            "Cannot add to expired lock. Withdraw"
        );

        _deposit_for(
            _tokenId,
            _value,
            0,
            _locked,
            DepositType.INCREASE_LOCK_AMOUNT
        );
    }
```

Recommendations:
Either remove the restriction on `increase_amount`, or add a restriction to `deposit_for`.

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
