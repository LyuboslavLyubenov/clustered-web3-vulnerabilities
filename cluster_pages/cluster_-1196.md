# Cluster -1196

**Rank:** #110  
**Count:** 154  

## Label
Inconsistent interface-implementation alignment leads to runtime errors, incorrect ABI calls, and security risks due to missing or mismatched function signatures, data types, and visibility.

## Cluster Information
- **Total Findings:** 154

## Examples

### Example 1

**Auto Label:** Inconsistent interface-implementation alignment leads to runtime errors, incorrect ABI calls, and security risks due to missing or mismatched function signatures, data types, and visibility.  

**Original Text Preview:**

[Pull request 1031](https://github.com/across-protocol/contracts/pull/1031/commits/e05964b074e6906e4dcb1d0fcd333dc7eb0b87be) removes already announced deprecated functionalities, such as the `depositDeprecated_5947912356` and `depositFor` functions, alongside their `_deposit` internal function.

However, as these entry points are removed, future versions of the codebase might introduce new functions that could have the same function selector as the removed ones. In such case, a protocol that might have used the deprecated functionalities could now call to the new ones with unexpected outcomes.

Similarly, if a `fallback` function is used in the future, depending on its implementation, it might take the calldata of protocols using the old deprecated functionalities with similar results.

In order to prevent the reuse of the deprecated function selectors, consider keeping the public functions' declaration, without their original definition, and reverting the calls to them.

***Update:** Resolved in [pull request #1048](https://github.com/across-protocol/contracts/pull/1048/commits/0e262bf992636db588e122f3e4f8399b1fdb6e4f) at commit `0e262bf`.*

---
### Example 2

**Auto Label:** Inconsistent interface-implementation alignment leads to runtime errors, incorrect ABI calls, and security risks due to missing or mismatched function signatures, data types, and visibility.  

**Original Text Preview:**

The `#[interface_id]` attribute is implemented as a [procedural macro](https://github.com/OpenZeppelin/rust-contracts-stylus/blob/5ed33dd88c3d74a0d0ec6cc0ff53f55a36a74c48/contracts-proc/src/interface_id.rs#L15) that can be added to a trait. This macro re-generates the trait source code with the addition of an `interface_id` function to produce its [ERC-165 identifier](https://eips.ethereum.org/EIPS/eip-165).

The [macro consumes any `#[selector]`](https://github.com/OpenZeppelin/rust-contracts-stylus/blob/5ed33dd88c3d74a0d0ec6cc0ff53f55a36a74c48/contracts-proc/src/interface_id.rs#L28-L29) attributes within the trait, allowing them to exist in traits. Within the Stylus SDK, the `#[selector]` attribute may only exist within `#[public]` blocks and does not provide logic for the macro within traits. However, there is no mechanism in place to enforce that trait implementations match the interface of traits with custom selectors.

As a consequence, developers must remember to add any custom selectors in trait implementations to correctly support the interface identifier, and there are no checks to ensure this. As a result, the ABI for a particular implementation may not match its trait, violating assumed invariants within Rust and potentially leading to failed contract integrations.

Consider refactoring the implementation of the `interface_id` macro to verify compatibility in custom selectors with respective trait implementations.

***Update:** Acknowledged, will resolve. The Contracts for Stylus team stated:*

> *We have recommended the Stylus SDK team to include the `#[interface_id]` functionality and the option to retrieve selectors of each method and auto-implement each router function to build inheritance this way (Doing so would allow compile-time checks on custom selector differences between traits and implementation). We will work with the team to add this functionality in future iterations.*

---
### Example 3

**Auto Label:** Failure to correctly implement interface support functions, leading to false negatives in interface detection and compromised interoperability, authorization, and protocol interactions.  

**Original Text Preview:**

The `supportsInterface` function in the `ERC721WithURIBuilderUpgradeable` contract overrides the IERC165 interface but fails to provide support for the necessary interface IDs(`IERC165`). This is not a good practice and may return incorrect results.

```solidity
    function supportsInterface(bytes4 interfaceId)
        public
        view
        virtual
        override(IERC165, ERC721EnumerableUpgradeable)
        returns (bool)
    {
        return ERC721EnumerableUpgradeable.supportsInterface(interfaceId);
    }
```

To mitigate this issue, change it to:

````solidity
    function supportsInterface(bytes4 interfaceId)
        public
        view
        virtual
        override(IERC165, ERC721EnumerableUpgradeable)
        returns (bool)
    {
        return ERC721EnumerableUpgradeable.supportsInterface(interfaceId) || super.supportsInterface(interfaceId);
    }

---
