# Cluster -1170

**Rank:** #215  
**Count:** 50  

## Label
Missing authorization and account validation allow attackers to bypass access controls, manipulate state, or execute unauthorized operations through improper instruction checks or flawed account handling.

## Cluster Information
- **Total Findings:** 50

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
### Example 3

**Auto Label:** Missing authorization and account validation allow attackers to bypass access controls, manipulate state, or execute unauthorized operations through improper instruction checks or flawed account handling.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-05-dodo-cross-chain-dex-judging/issues/633 

## Found by 
1337, EgisSecurity, Goran, HeckerTrieuTien, JuggerNaut, coin2own, dhank

### Summary

When users bridge assets to Solana, they specify a list of accounts and if accounts should be read-only (protected) versus writable (modifiable). This is a critical security feature that prevents unauthorized access to user funds. However, a data parsing bug in `AccountEncoder` systematically corrupts these permission settings, marking user accounts as writable when they should be protected as read-only. This affects 100% of Solana bridge transactions. As a consequence, potential unauthorized token transfers or account state changes can occur.

### Root Cause

Library `AccountEncoder` implements [decompressAccounts](https://github.com/sherlock-audit/2025-05-dodo-cross-chain-dex/blob/main/omni-chain-contracts/contracts/libraries/AccountEncoder.sol#L19) function which takes bytes array as input and parses account info out of it:
```solidity
    function decompressAccounts(bytes memory input) internal pure returns (Account[] memory accounts) {
        assembly {
              let ptr := add(input, 32)
  
              // Read accounts length (uint16)
              let len := add(shl(8, byte(0, mload(ptr))), byte(1, mload(ptr)))
              ptr := add(ptr, 2)
  
              // Allocate memory for Account[] array
              accounts := mload(0x40)
              mstore(accounts, len)
              let arrData := add(accounts, 32)

            // Prepare memory for Account structs
            let freePtr := add(arrData, mul(len, 32))

            for { let i := 0 } lt(i, len) { i := add(i, 1) } {
                let acc := freePtr
                freePtr := add(freePtr, 64)

                // Load publicKey
                mstore(acc, mload(ptr))
                ptr := add(ptr, 32)

                // Load isWritable (as bool)
                 // @audit Reads 32 bytes for 1-byte boolean
                mstore(add(acc, 32), iszero(iszero(mload(ptr))))
                ptr := add(ptr, 1)

                // Store pointer to struct in the array
                mstore(add(arrData, mul(i, 32)), acc)
            }

            mstore(0x40, freePtr)
        }
    }
```

In the assembly loop, the code reads 32 bytes with `mload(ptr)` when parsing boolean values that should only be 1 byte. This causes the boolean logic `iszero(iszero(...))` to evaluate the intended 1-byte boolean plus the first 31 bytes of the next account's public key. Since public keys contain non-zero bytes, the boolean always becomes true regardless of the user's intention.

### Internal Pre-conditions

1. User initiates Solana bridge transaction `(decoded.dstChainId == SOLANA_EDDY)`
2. Transaction contains multiple accounts in compressed format
3. Function `decompressAccounts()` is called to parse account metadata before Solana execution

### External Pre-conditions

None

### Attack Path

This is not an attack but systematic corruption affecting every multi-account Solana transaction:

1. User provides correctly formatted compressed account data: `[count][pubkey1][bool1][pubkey2][bool2]`
2. User intends Account 1 as read-only (bool1 = false) to protect their token balance
3. `decompressAccounts()` parses Account 1's boolean by reading `[bool1 + first_31_bytes_of_pubkey2]`
4. Since pubkey2 contains non-zero bytes, `iszero(iszero(non-zero))` returns true
5. Account 1 gets marked as `isWritable: true` instead of user's intended `false`
6. Transaction executes on Solana with corrupted permissions
7. Solana programs can now modify Account 1 when user expected it to remain untouched
8. Potential unauthorized token transfers or account state changes occur

### Impact

High severity - user accounts intended as read-only systematically get marked as writeable. This affects 100% of Solana bridge transactions with multiple accounts and occurs silently without any indication to users that their security assumptions are violated, potentially resulting in unexpected fund drains or account modifications.

### PoC

This test case showcases the problem:
```solidity
    function test_accounts() public {
        bytes32[] memory publicKeys = new bytes32[](2);
        publicKeys[0] = keccak256(abi.encodePacked(block.timestamp));
        publicKeys[1] = keccak256(abi.encodePacked(block.timestamp + 1));
        bool[] memory isWritables = new bool[](2);
        isWritables[0] = false;
        isWritables[1] = true;

        console.log("----User provided values:");

        console.logBytes32(publicKeys[0]);
        console.log(isWritables[0]);

        console.logBytes32(publicKeys[1]);
        console.log(isWritables[1]);

        bytes memory compressed = compressAccounts(publicKeys, isWritables);

        console.log("\n");
        console.log("(----Parsed values");

        console.logBytes32(AccountEncoder.decompressAccounts(compressed)[0].publicKey);
        console.log(AccountEncoder.decompressAccounts(compressed)[0].isWritable);

        console.logBytes32(AccountEncoder.decompressAccounts(compressed)[1].publicKey);
        console.log(AccountEncoder.decompressAccounts(compressed)[1].isWritable);
    }
```

Run it:
```bash
❯ forge test --mt test_accounts -vvv

Logs:
  ----User provided values:
  0xb335f44a2f5f0b13a7007be5f03a32df2fe6c27804d6e6ea21294eb0705e09b4
  false
  0x2186009e2f515bf15a0990ff378c583734095fd9ed9c46c3d15e33a8daf0a619
  true


  ----Parsed values
  0xb335f44a2f5f0b13a7007be5f03a32df2fe6c27804d6e6ea21294eb0705e09b4
  true
  0x2186009e2f515bf15a0990ff378c583734095fd9ed9c46c3d15e33a8daf0a619
  true
```

We can see that the "isWriteable" flag for the first account got parsed as `true` when it is actually `false`

### Mitigation

Extract only the first byte:
```solidity
mstore(add(acc, 32), shr(248, mload(ptr)))  
```


## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/Skyewwww/omni-chain-contracts/pull/31

---
