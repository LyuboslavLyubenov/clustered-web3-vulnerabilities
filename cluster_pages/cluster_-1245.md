# Cluster -1245

**Rank:** #249  
**Count:** 37  

## Label
Inconsistent or incorrect total supply tracking due to unit errors, flawed state updates, or misuse of supply variables, leading to inaccurate supply reporting, division-by-zero, or invalid state transitions.

## Cluster Information
- **Total Findings:** 37

## Examples

### Example 1

**Auto Label:** Supply inflation due to missing supply decrements during merges or deposits, violating invariant accounting and causing false token supply growth.  

**Original Text Preview:**

## Severity

**Impact:** Low

**Likelihood:** High

## Description

The `supply` state variable shows total amount of KITTEN locked in the VotingEscrow contract.

When merging two NFTs in the VotingEscrow contract, the `from` NFT's tokens (`value0`) are transferred to the `to` NFT. However, while the NFT is burnt, the `supply` isn't reduced before calling `_deposit_for`. Then, inside `_deposit_for`, the `supply` is increased again by `value0`.

The `supply` variable will be artificially inflated by `value0` each time a merge operation occurs. This overstatement is permanent as there's no corresponding decrement. It represents an internal accounting error within the contract.

```solidity
    function merge(uint _from, uint _to) external {
        ...
        locked[_from] = LockedBalance(0, 0);
        _checkpoint(_from, _locked0, LockedBalance(0, 0));
        _burn(_from);
        _deposit_for(_to, value0, end, _locked1, DepositType.MERGE_TYPE); // @audit No supply reduction before this call
    }
```

In the `_deposit_for` function, the supply is always increased:

```solidity
    function _deposit_for(
        uint _tokenId,
        uint _value,
        uint unlock_time,
        LockedBalance memory locked_balance,
        DepositType deposit_type
    ) internal {
        LockedBalance memory _locked = locked_balance;
        uint supply_before = supply;

        supply = supply_before + _value; // @audit original_total_supply_including_value0 + value0
        ...
    }
```

## Recommendations

Decrease the `supply` in the `merge` function before calling `_deposit_for`:

```diff
    function merge(uint _from, uint _to) external {
        ...
        locked[_from] = LockedBalance(0, 0);
        _checkpoint(_from, _locked0, LockedBalance(0, 0));
        _burn(_from);
+       supply -= value0;
        _deposit_for(_to, value0, end, _locked1, DepositType.MERGE_TYPE);
    }
```

---
### Example 2

**Auto Label:** Supply inflation due to missing supply decrements during merges or deposits, violating invariant accounting and causing false token supply growth.  

**Original Text Preview:**

## Severity

**Impact:** Low

**Likelihood:** High

## Description

The `supply` state variable shows total amount of KITTEN locked in the VotingEscrow contract.

When merging two NFTs in the VotingEscrow contract, the `from` NFT's tokens (`value0`) are transferred to the `to` NFT. However, while the NFT is burnt, the `supply` isn't reduced before calling `_deposit_for`. Then, inside `_deposit_for`, the `supply` is increased again by `value0`.

The `supply` variable will be artificially inflated by `value0` each time a merge operation occurs. This overstatement is permanent as there's no corresponding decrement. It represents an internal accounting error within the contract.

```solidity
    function merge(uint _from, uint _to) external {
        ...
        locked[_from] = LockedBalance(0, 0);
        _checkpoint(_from, _locked0, LockedBalance(0, 0));
        _burn(_from);
        _deposit_for(_to, value0, end, _locked1, DepositType.MERGE_TYPE); // @audit No supply reduction before this call
    }
```

In the `_deposit_for` function, the supply is always increased:

```solidity
    function _deposit_for(
        uint _tokenId,
        uint _value,
        uint unlock_time,
        LockedBalance memory locked_balance,
        DepositType deposit_type
    ) internal {
        LockedBalance memory _locked = locked_balance;
        uint supply_before = supply;

        supply = supply_before + _value; // @audit original_total_supply_including_value0 + value0
        ...
    }
```

## Recommendations

Decrease the `supply` in the `merge` function before calling `_deposit_for`:

```diff
    function merge(uint _from, uint _to) external {
        ...
        locked[_from] = LockedBalance(0, 0);
        _checkpoint(_from, _locked0, LockedBalance(0, 0));
        _burn(_from);
+       supply -= value0;
        _deposit_for(_to, value0, end, _locked1, DepositType.MERGE_TYPE);
    }
```

---
### Example 3

**Auto Label:** Supply inflation due to missing supply decrements during merges or deposits, violating invariant accounting and causing false token supply growth.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

The `GigaNameNFT `contract currently allows unlimited minting due to two key issues:

1. `confirmMint` lacks a `maxSupply `check: This function does not validate the total number of minted tokens against a supply limit, allowing infinite token creation.

2. `mint` function calls `GameNFT._safeMint` which assumes token IDs are sequential and represent the total supply, but this GigaNameNFT uses non-sequential IDs generated by hashing the username.

```solidity
    function _safeMint(address to, uint256 tokenId, bytes memory data) internal override {
        uint256 _maxSupply = maxSupply();
        if (_maxSupply != 0 && tokenId > _maxSupply) {
            revert TokenIdExceedsMaxSupply();
        }
```

3.  `GameNFT._safeMint` does not consider burned tokens for calculating the total supply

While the `maxSupply` is currently set to 0 (infinite supply), this is a potential issue for future versions where a finite `maxSupply` might be introduced. If not addressed, these issues could lead to over-minting or incorrect enforcement of supply limits.

## Recommendations

Introduce a `totalSupply` counter to accurately track minted tokens and enforce `maxSupply` in both `confirmMint` and `_safeMint` without assuming sequential token IDs and considering burned tokens.

---
