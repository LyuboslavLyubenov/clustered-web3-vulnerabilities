# Cluster -1006

**Rank:** #18  
**Count:** 734  

## Label
Inconsistent state updates between loan lifecycle contracts enable premature liquidations, bypassing payment timelines and default rules, allowing malicious actors to exploit timing and state immutability for low-cost attacks.

## Cluster Information
- **Total Findings:** 734

## Examples

### Example 1

**Auto Label:** Improper limit validation due to flawed state tracking, leading to unjustified rejections of stakes and asset locking.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

The `StakingManager` contract implements a staking limit mechanism through the `stakingLimit` parameter to control the maximum amount of HYPE tokens that can be staked in the system. However, the current implementation fails to accurately track the actual amount of tokens under management due to not accounting for withdrawn tokens.

The `stake()` function performs a limit check using:

```solidity
require(totalStaked + msg.value <= stakingLimit, "Staking limit reached");
```

However, `totalStaked` is only incremented when users stake and never decremented, while a separate `totalClaimed` tracks withdrawals. This means the actual amount of tokens under management is `totalStaked - totalClaimed`, but the limit check uses the raw `totalStaked` value.

This creates a situation where the contract could reject new stakes even when the actual amount under management is well below the limit, effectively reducing the protocol's capacity unnecessarily.

### Proof of Concept

- Admin sets stakingLimit to 1000 HYPE
- Alice stakes 1000 HYPE (totalStaked = 1000)
- Alice withdraws 500 HYPE (totalClaimed = 500, totalStaked = 1000)
- Bob tries to stake 100 HYPE
- Transaction reverts due to "Staking limit reached" even though only 500 HYPE is actually staked

## Recommendations

Update the staking limit check in `stake()` to account for claimed tokens:

```solidity
require(((totalStaked - totalClaimed) + msg.value) <= stakingLimit, "Staking limit reached");
```

---
### Example 2

**Auto Label:** Improper limit validation due to flawed state tracking, leading to unjustified rejections of stakes and asset locking.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

The `StakingManager` contract implements staking limits through three key parameters:
- `stakingLimit`: Maximum total stake allowed (0 = unlimited)
- `minStakeAmount`: Minimum stake per transaction
- `maxStakeAmount`: Maximum stake per transaction (0 = unlimited)

These parameters can be configured by accounts with the `MANAGER_ROLE` through dedicated setter functions. However, there is a logical error in the `setMaxStakeAmount()` function that prevents setting a valid max stake amount when there is no staking limit configured.

```solidity
        if (newMaxStakeAmount > 0) {
            require(newMaxStakeAmount > minStakeAmount, "Max stake must be greater than min");
            require(newMaxStakeAmount < stakingLimit, "Max stake must be less than limit");
        }
```

The issue occurs in the validation logic of `setMaxStakeAmount()`, where it unconditionally requires that any non-zero `newMaxStakeAmount` must be less than `stakingLimit`. This check fails to account for the case where `stakingLimit` is 0 (unlimited), making it impossible to set a max stake amount when no total limit exists. 

### Proof of Concept

1. Initially `stakingLimit = 0` (unlimited total staking)
2. Manager attempts to set `maxStakeAmount = 100 ether` to limit individual transactions
3. The call to `setMaxStakeAmount(100 ether)` reverts because:
   - `newMaxStakeAmount > 0` triggers the validation checks
   - `require(newMaxStakeAmount < stakingLimit)` fails as `100 ether < 0` is false

## Recommendations

Modify the `setMaxStakeAmount()` function to only perform the staking limit validation when a limit is actually set:

```solidity
function setMaxStakeAmount(uint256 newMaxStakeAmount) external onlyRole(MANAGER_ROLE) {
    if (newMaxStakeAmount > 0) {
        require(newMaxStakeAmount > minStakeAmount, "Max stake must be greater than min");
        if (stakingLimit > 0) {
            require(newMaxStakeAmount < stakingLimit, "Max stake must be less than limit");
        }
    }
    maxStakeAmount = newMaxStakeAmount;
    emit MaxStakeAmountUpdated(newMaxStakeAmount);
}
```

---
### Example 3

**Auto Label:** Flawed state synchronization and improper validation lead to incorrect asset balances, misrouted fees, and unauthorized access, enabling attacks, DoS, and economic exploitation.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** High

## Description

Due to the lack of a check on the split amount inside `split`, a user can provide an arbitrary `_amount` to the function. Since `locked.amount` is stored as an `int128`, this can result in one of the split tokens having a negative value, while the other is set to the full `_amount` provided.

```solidity
    function split(
        uint _from,
        uint _amount
    ) external returns (uint _tokenId1, uint _tokenId2) {
        address msgSender = msg.sender;

        require(_isApprovedOrOwner(msgSender, _from));
        require(attachments[_from] == 0 && !voted[_from], "attached");
        require(_amount > 0, "Zero Split");

        // burn old NFT
        LockedBalance memory _locked = locked[_from];
        int128 value = _locked.amount;
        locked[_from] = LockedBalance(0, 0);
        _checkpoint(_from, _locked, LockedBalance(0, 0));
        _burn(_from);

        // set max lock on new NFTs
        _locked.end = block.timestamp + MAXTIME;

        int128 _splitAmount = int128(uint128(_amount));
        // @audit - underflow is possible
>>>     _locked.amount = value - _splitAmount; // already checks for underflow here in ^0.8.0
        _tokenId1 = _createSplitNFT(msgSender, _locked);

>>>     _locked.amount = _splitAmount;
        _tokenId2 = _createSplitNFT(msgSender, _locked);

        emit Split(
            _from,
            _tokenId1,
            _tokenId2,
            msgSender,
            uint(uint128(locked[_tokenId1].amount)),
            uint(uint128(_splitAmount)),
            _locked.end,
            block.timestamp
        );
    }
```

This can be abused by initially depositing a small amount of tokens, then calling `split` with an arbitrary amount to create a large `locked` data. Then can be withdrawn at the end of the lock period, and also grants an arbitrary amount of voting power and balance, which can be used to claim rewards from gauges and bribes.

PoC :

```solidity
    function testSplitVeKittenUnderflow() public {
        testDistributeVeKitten();

        vm.startPrank(user1);

        uint veKittenId = veKitten.tokenOfOwnerByIndex(user1, 0);
        (int128 lockedAmount, uint endTime) = veKitten.locked(veKittenId);

        (uint t1, uint t2) = veKitten.split(veKittenId, uint256(uint128(lockedAmount)) * 100);

        (int128 lockedAmountSplit, ) = veKitten.locked(t2);

        console.log("initial token to split locked amount :");
        console.log(lockedAmount);
        console.log("splitted token locked amount :");
        console.log(lockedAmountSplit);

        vm.stopPrank();
    }
```

Log output :

```shell
Logs:
  initial token to split locked amount :
  100000000000000000000000000
  splitted token locked amount :
  10000000000000000000000000000
```

## Recommendations

Validate that the provided `_amount` does not exceed the locked amount of the split token.

---
