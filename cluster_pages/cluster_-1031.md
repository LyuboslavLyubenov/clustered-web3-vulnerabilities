# Cluster -1031

**Rank:** #295  
**Count:** 25  

## Label
Redundant and misapplied access checks lead to unreachable requirements, increased code duplication, and inconsistent authorization, resulting in denial-of-service or unexpected behavior.

## Cluster Information
- **Total Findings:** 25

## Examples

### Example 1

**Auto Label:** Redundant access controls in view functions lead to unnecessary complexity, increased code confusion, and potential security misalignments without functional impact.  

**Original Text Preview:**

##### Description
The issue is identified within the [`getOrderInfo`](https://github.com/longgammalabs/hanji-contracts/blob/09b6188e028650b9c1758010846080c5f8c80f8e/src/TrieLib.sol#L155) function of the `Trie` contract.

This function has a redundant `onlyLOBPermission` modifier. Since this is a view function, it should not require restrictive permissions. This redundancy complicates the code without adding necessary security or functionality.
##### Recommendation
We recommend removing the `onlyLOBPermission` modifier from the `getOrderInfo` function to simplify it. This will enhance code readability and maintainability without compromising security.

---
### Example 2

**Auto Label:** Redundant access controls in view functions lead to unnecessary complexity, increased code confusion, and potential security misalignments without functional impact.  

**Original Text Preview:**

##### Description
The issue is identified within the [`getOrderInfo`](https://github.com/longgammalabs/hanji-contracts/blob/09b6188e028650b9c1758010846080c5f8c80f8e/src/TrieLib.sol#L155) function of the `Trie` contract.

This function has a redundant `onlyLOBPermission` modifier. Since this is a view function, it should not require restrictive permissions. This redundancy complicates the code without adding necessary security or functionality.
##### Recommendation
We recommend removing the `onlyLOBPermission` modifier from the `getOrderInfo` function to simplify it. This will enhance code readability and maintainability without compromising security.

---
### Example 3

**Auto Label:** Redundant access controls in view functions lead to unnecessary complexity, increased code confusion, and potential security misalignments without functional impact.  

**Original Text Preview:**

##### Description
The issue is identified within the [`getOrderInfo`](https://github.com/longgammalabs/hanji-contracts/blob/09b6188e028650b9c1758010846080c5f8c80f8e/src/TrieLib.sol#L155) function of the `Trie` contract.

This function has a redundant `onlyLOBPermission` modifier. Since this is a view function, it should not require restrictive permissions. This redundancy complicates the code without adding necessary security or functionality.
##### Recommendation
We recommend removing the `onlyLOBPermission` modifier from the `getOrderInfo` function to simplify it. This will enhance code readability and maintainability without compromising security.

---
