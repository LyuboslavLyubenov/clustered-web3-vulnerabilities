# Cluster -1303

**Rank:** #236  
**Count:** 41  

## Label
Buffer overflows and memory safety flaws due to inadequate bounds checking, improper input validation, and unsafe memory access patterns, leading to data corruption, undefined behavior, or exposure of sensitive information.

## Cluster Information
- **Total Findings:** 41

## Examples

### Example 1

**Auto Label:** Buffer overflows and memory safety flaws due to inadequate bounds checking, improper input validation, and unsafe memory access patterns, leading to data corruption, undefined behavior, or exposure of sensitive information.  

**Original Text Preview:**

The [`from_str_hex`](https://github.com/OpenZeppelin/rust-contracts-stylus/blob/a67ab7068ac94a5a1ad48ae632050c8d28f9ab25/lib/crypto/src/arithmetic/uint.rs#L745) function does not include an overflow check when parsing hexadecimal strings into a `Uint<LIMBS>` type. This omission may result in an unexpected index out-of-bounds panic if the input string represents a number exceeding the capacity of the `Uint<LIMBS>` type. In contrast, the `from_str_radix` function incorporates an [overflow check](https://github.com/OpenZeppelin/rust-contracts-stylus/blob/a67ab7068ac94a5a1ad48ae632050c8d28f9ab25/lib/crypto/src/arithmetic/uint.rs#L722) within the [`ct_add`](https://github.com/OpenZeppelin/rust-contracts-stylus/blob/a67ab7068ac94a5a1ad48ae632050c8d28f9ab25/lib/crypto/src/arithmetic/uint.rs#L393) function, ensuring that overflow conditions are explicitly detected and trigger a panic with a clear error message.

Consider implementing an overflow check in the `from_str_hex` function to enhance its robustness.

***Update:** Resolved in [pull #673](https://github.com/OpenZeppelin/rust-contracts-stylus/pull/673/files) at comit [a988e2b](https://github.com/OpenZeppelin/rust-contracts-stylus/commit/a988e2bdd9f1e86b9dd8fcf8a5bc77c3b6ef16e6).*

---
### Example 2

**Auto Label:** Insufficient bounds checking and improper memory management lead to memory overflows, out-of-bounds access, and double-free errors, enabling unauthorized memory corruption and undefined behavior.  

**Original Text Preview:**

In both [`modexpGasCost`](https://github.com/matter-labs/era-contracts/blob/cc1619cfb03cc19adb21a2071c89415cab1479e8/system-contracts/evm-emulator/EvmEmulatorFunctions.template.yul#L985) and [`mloadPotentiallyPaddedValue`](https://github.com/matter-labs/era-contracts/blob/cc1619cfb03cc19adb21a2071c89415cab1479e8/system-contracts/evm-emulator/EvmEmulatorFunctions.template.yul#L1052) functions, the code calculates shift amounts in bytes but uses them directly with EVM shift instructions, which operate on bits. This results in incomplete or inaccurate shifts, as 1 byte equals 8 bits. Without converting byte-based values into their bit equivalents, the shift operations behave incorrectly.

This mismatch has several negative effects:

* **Distorted Parameter Reads**: When used to trim or isolate components like the base, exponent, or modulus, incorrect shifts may leave behind unintended bits. This can lead to inflated sizes, skewed gas calculations, or corrupted numerical values.
* **Exploitability Risk**: Malicious users may supply inputs that trigger these incorrect shifts, potentially manipulating gas costs or bypassing boundary checks, leading to undefined or exploitable behavior.
* **Incorrect Memory Interpretation**: Code paths intended to mask or sanitize specific bytes may instead leave residual bits intact. This can cause logical errors when interpreting memory content.
* **Numerical Instability**: Misaligned shift results can cause values to overflow or underflow in downstream logic. For instance, malformed bit-lengths derived from exponent parsing may cause loops to run excessively or insufficiently.

Consider ensuring that every shift amount derived from a byte difference is multiplied by 8 before applying any shift operation. This guarantees alignment with EVM's bit-level shift semantics and avoids the wide range of downstream issues stemming from partial shifts.

***Update:** Resolved in [pull request #1383](https://github.com/matter-labs/era-contracts/pull/1383).*

---
### Example 3

**Auto Label:** Insufficient bounds checking and improper memory management lead to memory overflows, out-of-bounds access, and double-free errors, enabling unauthorized memory corruption and undefined behavior.  

**Original Text Preview:**

##### Description
The issue has been identified within the [freeMemory](https://github.com/longgammalabs/hanji-contracts/blob/70b15ec4d9e7578248141604503843716a67d875/src/HanjiTrie.sol#L1274) function of the `HanjiTrie` contract.

Currently, there is no check to verify whether a pointer has already been freed before the `freeMemory()` function is called. Although the current implementation does not have any double `freeMemory()` calls or other destructive actions involving the `arena` mapping, future modifications to the codebase could introduce such risks. If a pointer is freed more than once, the `last_free_pointer` variable may be set to the index of `arena` element where the `last_free_pointer` is also stored, causing the `allocateMemory()` function to repeatedly return the same pointer, leading to potential memory corruption or unexpected behavior.

The issue is classified as **low** severity because it does not pose an immediate risk but could result in critical errors or vulnerabilities if the code is modified in the future.
##### Recommendation
We recommend adding a check in the `freeMemory()` function to ensure that the pointer being freed is valid and has not already been freed.

---
