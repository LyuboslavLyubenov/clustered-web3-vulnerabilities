# Cluster -1276

**Rank:** #250  
**Count:** 37  

## Label
Inadequate calldata validation enables attackers to manipulate input lengths or structures, leading to unintended function execution, malformed decoding, or denial of service through invalid or padded payloads.

## Cluster Information
- **Total Findings:** 37

## Examples

### Example 1

**Auto Label:** Inadequate calldata validation enables attackers to manipulate input lengths or structures, leading to unintended function execution, malformed decoding, or denial of service through invalid or padded payloads.  

**Original Text Preview:**

In order to determine the CCTP version which will be used for bridging USDC, the constructor of the [`CircleCCTPAdapter` contract](https://github.com/across-protocol/contracts/blob/0720878762ebf99034aa62893c091237ef44fdf7/contracts/libraries/CircleCCTPAdapter.sol#L28) performs a [low level call](https://github.com/across-protocol/contracts/blob/0720878762ebf99034aa62893c091237ef44fdf7/contracts/libraries/CircleCCTPAdapter.sol#L83-L85) to the `feeRecipient` function of a CCTP TokenMessenger. The check relies on the fact that this function is not present in V1 contracts, but is present in V2 contracts and returns a non-zero address.

However, the check being performed is not fully reliable for the reasons enumerated below.

The values returned by the functions in Solidity are `abi.encode`d, which means that the address returned by the `feeRecipient` function will be padded to 32 bytes with 0s at the beginning. Furthermore, casting a `bytes` object to `bytes20` will return the first 20 bytes of that object, which will include 12 bytes of padding and as a result, only the first 8 bytes of the returned address will be taken into account in [this check](https://github.com/across-protocol/contracts/blob/0720878762ebf99034aa62893c091237ef44fdf7/contracts/libraries/CircleCCTPAdapter.sol#L86C30-L86C74). In case when the first 8 bytes of the returned `feeRecipient` are equal to 0, the contract will incorrectly assume that it should use CCTP V1.

Furthermore, while it is checked that the call succeeded, there is no verification that the return data is of correct size. As a result, the [check](https://github.com/across-protocol/contracts/blob/0720878762ebf99034aa62893c091237ef44fdf7/contracts/libraries/CircleCCTPAdapter.sol#L86C30-L86C74) can succeed in case when the return data is of a different size. In particular, for the return data of 0 length, the correctness of this check relies on the cast from an empty `bytes` object to `bytes20` always returning 0.

Finally, a contract to be called could still have a fallback function which returns 32 bytes of data. In case when at least 1 of the last 20 bytes of that data is nonzero, the check will still succeed even if the target contract does not implement the `feeRecipient` function.

Consider casting the `feeRecipient` to `uint160` before casting it to address in order to retrieve the full address. Moreover, consider validating the length of the data returned by the low level call in order to better handle the situations when data with unexpected size is returned. Furthermore, consider implementing a more robust mechanism of determining the CCTP version to be used.

***Update:** Partially resolved in [pull request #921](https://github.com/across-protocol/contracts/pull/921) at commit [d9d4707](https://github.com/across-protocol/contracts/pull/921/commits/d9d4707d8b19d2d2bd80632ed411a22ef031b0dc). The casting of the data returned from a low level call has been fixed and additional verification has been added to the length of the returned data. It is still possible that the check for the CCTP version will not determine the correct version, but this scenario is very unlikely and the risk has been accepted by the Risk Labs team.*

---
### Example 2

**Auto Label:** Inadequate calldata validation enables attackers to manipulate input lengths or structures, leading to unintended function execution, malformed decoding, or denial of service through invalid or padded payloads.  

**Original Text Preview:**

##### Description
The `_targetContract` function extracts the `address` of the vault contract from the calldata by assuming the last 20 bytes represent the address. While there's a check that `msg.data.length > 20`, this check allows possible manipulations of the calldata which can bring additional attack vectors in the future. This function requires exactly 88 bytes to process the call (4 bytes (selector) + 32 (uint16) + 32 (uint16) + 20 (address)).

This opens a path for potential calldata manipulation or malformed call attempts, especially when relying on calldata decoding for internal logic.
<br/>
##### Recommendation
Enhance the validation logic by enforcing strict size:
```solidity
if (msg.data.length != 88) revert MsgDataInvalid();
```

> **Client's Commentary:**
> Acknowledge, won't fix. This attack vector is theoretical and can only be realized in a highly specific setup. Furthermore, state-changing functions in CapRiskManager are behind access control mechanisms, so the caller is assumed to be trusted.

---
### Example 3

**Auto Label:** Inadequate calldata validation enables attackers to manipulate input lengths or structures, leading to unintended function execution, malformed decoding, or denial of service through invalid or padded payloads.  

**Original Text Preview:**

The `checkAndUpdate` function incorrectly extracts calldata parameters when handling dynamic types larger than 32 bytes. The function assumes all parameters can be extracted using a fixed 32-byte offset, but this fails for dynamic types that use a more complex encoding scheme.

```solidity
  function checkAndUpdate(
    Constraint memory constraint,
    UsageTracker storage tracker,
    bytes calldata data,
    uint64 period
  ) internal {
    uint256 index = 4 + constraint.index * 32;
    bytes32 param = bytes32(data[index:index + 32]); // @audit Incorrect assumption that parameter can be extracted with fixed offset
    Condition condition = constraint.condition;
    bytes32 refValue = constraint.refValue;

    ...
  }
```

Let's consider the following example:

This is the function signature:

```solidity
contract Example {
    function exampleFunction(string memory text, uint256[] memory numbers) external {}
}
```

There are two cases:

1.  First case when the text is less than 32 bytes - hello word is included in 32-byte word.

Input: `hello`, [1,2,3]

```text
- 0x303430f7                                                       // function signature
- 0000000000000000000000000000000000000000000000000000000000000040 // offset for text
- 0000000000000000000000000000000000000000000000000000000000000080 // offset for numbers
- 0000000000000000000000000000000000000000000000000000000000000005 // length of string
- 68656c6c6f000000000000000000000000000000000000000000000000000000 // hello
- 0000000000000000000000000000000000000000000000000000000000000003 // length of the array
- 0000000000000000000000000000000000000000000000000000000000000001 // 1
- 0000000000000000000000000000000000000000000000000000000000000002 // 2
- 0000000000000000000000000000000000000000000000000000000000000003 // 3
```

2. Second case when the text is more than 32 bytes - The data spans multiple 32-byte words.

Input: `helloooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo`, [1,2,3]

```text
- 0x303430f7                                                       // function signature
- 0000000000000000000000000000000000000000000000000000000000000040 // offset for text
- 00000000000000000000000000000000000000000000000000000000000000c0 // offset for numbers
- 0000000000000000000000000000000000000000000000000000000000000048 // length of string
- 68656c6c6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f // text
- 6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f6f // text
- 6f6f6f6f6f6f6f6f000000000000000000000000000000000000000000000000 // text
- 0000000000000000000000000000000000000000000000000000000000000003 // length of the array
- 0000000000000000000000000000000000000000000000000000000000000001 // 1
- 0000000000000000000000000000000000000000000000000000000000000002 // 2
- 0000000000000000000000000000000000000000000000000000000000000003 // 3
```

In the second case, the text cannot be extracted correctly using one index like in `checkAndUpdate` function. It leads to misinterpretation of the actual parameter values with the following impact:

- Session key constraints can be bypassed
- Functions with dynamic parameters become unusable with session keys when constraints are set on the dynamic parameters.

Handle the dynamic types correctly in the `checkAndUpdate` function using multiple 32-byte words.

---
