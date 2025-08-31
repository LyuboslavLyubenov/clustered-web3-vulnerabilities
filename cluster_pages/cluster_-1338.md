# Cluster -1338

**Rank:** #83  
**Count:** 210  

## Label
Failure to validate permit signatures leads to front-running, phishing, and unauthorized transfers due to insufficient message transparency, allowance checks, and nonce enforcement.

## Cluster Information
- **Total Findings:** 210

## Examples

### Example 1

**Auto Label:** Failure to properly bind signatures to chain ID and contract context enables replay attacks, signature reuse, and incorrect state validation across chains and transactions.  

**Original Text Preview:**

The [`deposit` function](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L189) of the `SpokePoolPeriphery` contract allows users to deposit native value to the SpokePool. However, its `recipient` and `exclusiveRelayer` arguments are both of type `address` and are [cast](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L209-L215) to `bytes32`. As a result, it is not possible to bridge wrapped native tokens to non-EVM blockchains.

Consider changing the type of the `recipient` and `exclusiveRelayer` arguments of the `deposit` function so that callers are allowed to specify non-EVM addresses for deposits.

***Update:** Resolved in [pull request #1018](https://github.com/across-protocol/contracts/pull/1018) at commit [`3f34af6`](https://github.com/across-protocol/contracts/pull/1018/commits/3f34af68b7602873e59a5a54b7c9cb0982f49d0e).*

---
### Example 2

**Auto Label:** Failure to properly bind signatures to chain ID and contract context enables replay attacks, signature reuse, and incorrect state validation across chains and transactions.  

**Original Text Preview:**

The [`PeripherySigningLib` library](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/libraries/PeripherySigningLib.sol#L6) contains the EIP-712 encodings of certain types as well as helper functions to generate their EIP-712 compliant hashed data. However, the [data type of the `SwapAndDepositData` struct](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/libraries/PeripherySigningLib.sol#L12-L13) is incorrect as it contains the `TransferType` member [of an enum type](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/interfaces/SpokePoolPeripheryInterface.sol#L18-L25), which is not supported by the EIP-712 standard.

Consider replacing the `TransferType` enum name used to generate the `SwapAndDepositData` struct's data type with `uint8` in order to be compliant with EIP-712.

***Update:** Resolved in [pull request #1017](https://github.com/across-protocol/contracts/pull/1017) at commit [`c9aaec6`](https://github.com/across-protocol/contracts/pull/1017/commits/c9aaec6d26993314e0bf878cd4eb89f447194789).*

---
### Example 3

**Auto Label:** Failure to properly bind signatures to chain ID and contract context enables replay attacks, signature reuse, and incorrect state validation across chains and transactions.  

**Original Text Preview:**

<https://github.com/code-423n4/2025-05-blackhole/blob/92fff849d3b266e609e6d63478c4164d9f608e91/contracts/VotingEscrow.sol# L1205>

<https://github.com/code-423n4/2025-05-blackhole/blob/92fff849d3b266e609e6d63478c4164d9f608e91/contracts/VotingEscrow.sol# L1327-L1335>

### Finding description

There is a mistake in how the `DOMAIN_TYPEHASH` constant is defined and used in the `VotingEscrow` contract.
```

bytes32 public constant DOMAIN_TYPEHASH = keccak256("EIP712Domain(string name,uint256 chainId,address verifyingContract)");
```

However, when building the domain separator, the contract includes an additional parameter:
```

bytes32 domainSeparator = keccak256(
    abi.encode(
        DOMAIN_TYPEHASH,
        keccak256(bytes(name)),
        keccak256(bytes(version)),
        block.chainid,
        address(this)
    )
);
```

The problem is that the `DOMAIN_TYPEHASH` does **not** include the `version` parameter, but the contract still tries to encode it. This creates a mismatch between the type hash and the actual encoding, which will lead to incorrect `digest` hashes when signing or verifying messages.

### Impact

* Users will be unable to sign or verify messages using the EIP-712 delegation feature.
* Delegation by signature (`delegateBySig`) will always fail due to signature mismatch.
* Governance features relying on off-chain signatures will break.

### Recommended mitigation steps

Update the `DOMAIN_TYPEHASH` to include the `version` field so that it matches the data structure used in the actual `domainSeparator`:
```

bytes32 public constant DOMAIN_TYPEHASH = keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)");
```

This change ensures the type hash includes all the fields being encoded and fixes the signature validation logic.

**Blackhole marked as informative**

---

---
