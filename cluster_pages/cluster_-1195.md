# Cluster -1195

**Rank:** #194  
**Count:** 58  

## Label
Hash collisions from improper ABI encoding due to lack of type boundaries, enabling signature forgery, unauthorized claims, and misrouted callbacks through malleable, non-deterministic byte concatenation.

## Cluster Information
- **Total Findings:** 58

## Examples

### Example 1

**Auto Label:** Lack of compile-time type safety in ABI encoding leads to function signature mismatches, causing incorrect function calls, reentrancy, and exploitable runtime errors.  

**Original Text Preview:**

The `fill` [function](https://github.com/across-protocol/contracts/blob/108be77c29a3861c64bdf66209ac6735a6a87090/contracts/SpokePool.sol#L988) of the `SpokePool` contract is meant to adhere to the `IDestinationSettler` interface, as dictated by the latest update to the `ERC-7683` [specifications](https://github.com/across-protocol/ERCs/blob/d975d7b4b58fa3d1aa6db1763935cfa2ab1444b1/ERCS/erc-7683.md). The `fill` function is meant to internally call the `fillV3Relay` [function](https://github.com/across-protocol/contracts/blob/108be77c29a3861c64bdf66209ac6735a6a87090/contracts/SpokePool.sol#L864) in order to process the order data, and it does so by [making a `delegatecall` to its own `fillV3Relay` function](https://github.com/across-protocol/contracts/blob/108be77c29a3861c64bdf66209ac6735a6a87090/contracts/SpokePool.sol#L998-L999), passing `abi.encodePacked(originData, fillerData)` as the parameter.

However, the `fillV3Relay` function accepts two parameters, having the `repaymentChainId` as the second parameter. Since the call is constructed using `encodeWithSelector`, which is not type-safe, the compiler does not complain about the missing parameter. As an incorrect number of parameters is passed, the call to `fillV3Relay` will always revert when trying to decode the input parameters, breaking the entire execution flow. Moreover, the input data is encoded with `abi.encodePacked` the use of which is discouraged, especially when dealing with structs and dynamic types like arrays.

Consider using `encodeCall` instead of `encodeWithSelector` to ensure type safety, and providing the parameters required by the `fillV3Relay` function separately. In addition, consider explicitly making the `SpokePool` contract inherit from the `IDestinationSettler` interface as required by the `ERC-7683` standard.

***Update:** Resolved in [pull request #744](https://github.com/across-protocol/contracts/pull/744) at commit [9f54455](https://github.com/across-protocol/contracts/pull/744/commits/9f5445571a98f13248a21acba3ac3fe40c737abd).*

---
### Example 2

**Auto Label:** Hash collisions from improper ABI encoding due to lack of type boundaries, enabling signature forgery, unauthorized claims, and misrouted callbacks through malleable, non-deterministic byte concatenation.  

**Original Text Preview:**

...

Export to GitHub ...

Set external GitHub Repo ...

Export to Clipboard (json)

Export to Clipboard (text)

#### Resolution

Fixed in commit [5befce72aceb8851895e6cdcbae63d0ec47ef67a](https://github.com/MetaMask/delegation-framework/commit/5befce72aceb8851895e6cdcbae63d0ec47ef67a) by implementing `abi.encode()` instead of `abi.encodePacked()`.

#### Description

The contract uses `abi.encodePacked()` with dynamic types in hashing operations. This pattern is discouraged because `abi.encodePacked()` does not include length information for dynamic types, making it vulnerable to hash collisions. As a result, different sets of input data may produce identical hashes, potentially leading to critical security issues. This is especially problematic when passing the result to `keccak256()`, as it can undermine the uniqueness and integrity of signature verifications and other sensitive logic. The `_validateSignature` function is susceptible to a hash collision, during the encoding of `_signatureData.apiData` and `_signatureData.expiration` variables; however, due to a lack information, it’s impossible to create a real scenario. Still, it is recommended to change the type of encoding to be safe.

#### Examples

**src/helpers/DelegationMetaSwapAdapter.sol:L526-L534**

```
function _validateSignature(SignatureData memory _signatureData) private view {
    if (block.timestamp > _signatureData.expiration) revert SignatureExpired();

    bytes32 messageHash_ = keccak256(abi.encodePacked(_signatureData.apiData, _signatureData.expiration));
    bytes32 ethSignedMessageHash_ = MessageHashUtils.toEthSignedMessageHash(messageHash_);

    address recoveredSigner_ = ECDSA.recover(ethSignedMessageHash_, _signatureData.signature);
    if (recoveredSigner_ != swapApiSigner) revert InvalidApiSignature();
}

```

#### Recommendation

We recommend replacing `abi.encodePacked()` with `abi.encode()` when hashing multiple dynamic types to avoid potential collisions and ensure consistency.

---
### Example 3

**Auto Label:** Lack of compile-time type safety in ABI encoding leads to function signature mismatches, causing incorrect function calls, reentrancy, and exploitable runtime errors.  

**Original Text Preview:**

The first argument in the `DynamicAccount._validateSignature` function is currently of type `UserOperation`. However, according to the expected signature in `BaseAccount`, ([link](https://github.com/thirdweb-dev/contracts/blob/389f9456571fe554d7a048d34806cbbe7b3ec909/contracts/prebuilts/account/utils/BaseAccount.sol#L68)) the first argument should be `PackedUserOperation`. This misalignment in function arguments could revert the transaction.

```solidity
    function _validateSignature(
        UserOperation calldata userOp,
        bytes32 userOpHash
    ) internal virtual override returns (uint256 validationData) {
    --snip--
    }
```

Recommendations:

```solidity
function _validateSignature(
        PackedUserOperation calldata userOp,
        bytes32 userOpHash
    )
```

---
