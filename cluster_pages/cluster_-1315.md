# Cluster -1315

**Rank:** #231  
**Count:** 43  

## Label
Use of an experimental, unstable ABI encoding feature leads to undefined behavior, enabling attackers to manipulate contract state through type mismatches or incorrect data serialization.

## Cluster Information
- **Total Findings:** 43

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

**Auto Label:** Unbounded memory access via malformed input during ABI decoding, enabling memory corruption and arbitrary data injection.  

**Original Text Preview:**

## Security Analysis Report

## Difficulty
Not Applicable

## Type
Cryptography

## Description
Disabling validation when decoding calldata can produce undefined behavior when dealing with ERC-20 transactions. To perform a deposit into the Serai DEX, Ethereum users have multiple options. One involves sending an ERC-20 transaction using certain tokens directly to the router. Such a transaction will have extended calldata encoding the specific information for the deposit, called `InInstruction`.

The Ethereum/ERC-20 processor code iterates over all of the confirmed transactions, locating top-level ERC-20 transactions that can be converted to `InInstructions` deposits. Once a candidate transaction is located, its calldata is decoded, first with the original ERC-20 and then with an alternative ABI called `SeraiIERC20` that includes the encoded `InInstruction` value as a list of bytes.

```rust
async fn top_level_transfer(
    provider: &RootProvider<SimpleRequest>,
    erc20: Address,
    transaction_hash: [u8; 32],
    transfer_logs: &[Log],
) -> Result<Option<TopLevelTransfer>, RpcError<TransportErrorKind>> {
    ...
    // Read the data appended after
    let data = if let Ok(call) =
        SeraiIERC20Calls::abi_decode(transaction.inner.input(), true) {
        match call {
            SeraiIERC20Calls::transferWithInInstruction01BB244A8A(
                transferWithInInstructionCall { inInstruction, .. },
            ) |
            SeraiIERC20Calls::transferFromWithInInstruction00081948E0(
                transferFromWithInInstructionCall { inInstruction, .. },
            ) => Vec::from(inInstruction),
        }
    }
}
```

*Figure 1.1: Part of the `top_level_transfer` that decodes ABI from untrusted data*

However, an upcoming release of the `alloy-sol-types` library will remove the validation flag. This can be an issue with the current code, as disabling the validation for parsing the `SeraiIERC20` ABI can produce unexpected results, such as this crash:

```
---- tests::in_instruction::test_erc20_top_level_transfer_in_instruction stdout ----
thread 'tests::in_instruction::test_erc20_top_level_transfer_in_instruction' panicked at
/Users/g/.cargo/registry/src/index.crates.io-6f17d22bba15001f/alloy-sol-types-0.8.16/src/abi/decoder.rs:173:27: attempt to add with overflow
stack backtrace:
0: rust_begin_unwind
at /rustc/eeb90cda1969383f56a2637cbd3037bdf598841c/library/std/src/panicking.rs:665:5
1: core::panicking::panic_fmt
at /rustc/eeb90cda1969383f56a2637cbd3037bdf598841c/library/core/src/panicking.rs:74:14
2: core::panicking::panic_const::panic_const_add_overflow
at /rustc/eeb90cda1969383f56a2637cbd3037bdf598841c/library/core/src/panicking.rs:181:21
3: alloy_sol_types::abi::decoder::Decoder::peek_len_at
at /Users/g/.cargo/registry/src/index.crates.io-6f17d22bba15001f/alloy-sol-types-0.8.16/src/abi/decoder.rs:173:27
4: alloy_sol_types::abi::decoder::Decoder::peek_len
at /Users/g/.cargo/registry/src/index.crates.io-6f17d22bba15001f/alloy-sol-types-0.8.16/src/abi/decoder.rs:179:9
5: <alloy_sol_types::abi::token::PackedSeqToken as alloy_sol_types::abi::token::Token>::decode_from
at /Users/g/.cargo/registry/src/index.crates.io-6f17d22bba15001f/alloy-sol-types-0.8.16/src/abi/token.rs:473:21
6: <(T1,T2,T3) as alloy_sol_types::abi::token::TokenSeq>::decode_sequence
at /Users/g/.cargo/registry/src/index.crates.io-6f17d22bba15001f/alloy-sol-types-0.8.16/src/abi/token.rs:606:27
7: alloy_sol_types::abi::decoder::Decoder::decode_sequence
at /Users/g/.cargo/registry/src/index.crates.io-6f17d22bba15001f/alloy-sol-types-0.8.16/src/abi/decoder.rs:272:9
8: alloy_sol_types::abi::decoder::decode_sequence
at /Users/g/.cargo/registry/src/index.crates.io-6f17d22bba15001f/alloy-sol-types-0.8.16/src/abi/decoder.rs:321:18
9: alloy_sol_types::types::ty::SolType::abi_decode_sequence
at /Users/g/.cargo/registry/src/index.crates.io-6f17d22bba15001f/alloy-sol-types-0.8.16/src/types/ty.rs:274:9
10: alloy_sol_types::types::function::SolCall::abi_decode_raw
...
```

*Figure 1.2: A crash exposed when the validation flag is set to false*

Normally, this type of crash will be unreachable if the calldata is coming from confirmed transactions, but in this case, the ABI is never validated on-chain, and the crash can therefore be triggered externally.

## Exploit Scenario
The `alloy-sol-types` library is upgraded, introducing a bug that allows any user to remotely crash the processor.

## Recommendations
- **Short term**: Do not upgrade the `alloy-sol-types` library until you are certain that it is robust enough to handle arbitrary inputs.
- **Long term**: Review the security properties and risks introduced by the usage of third-party components across the codebase.

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
