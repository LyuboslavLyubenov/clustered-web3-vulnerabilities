# Cluster -1277

**Rank:** #156  
**Count:** 88  

## Label
Lack of immutability in state variables leads to unnecessary gas costs, compromised contract integrity, and potential runtime manipulation, undermining security and efficiency.

## Cluster Information
- **Total Findings:** 88

## Examples

### Example 1

**Auto Label:** Lack of immutability in state variables leads to unnecessary gas costs, compromised contract integrity, and potential runtime manipulation, undermining security and efficiency.  

**Original Text Preview:**

##### Description

The issue is identified within the [HanjiLOB](https://github.com/longgammalabs/hanji-contracts/blob/70b15ec4d9e7578248141604503843716a67d875/src/HanjiLOB.sol#L55) contract.

Variables with the `immutable` modifier do not use storage, which makes them more gas-efficient. In the context of a proxy pattern, it is still acceptable to initialize such variables in the constructor instead of the `initialize` function. Doing so can result in significant gas savings when these variables are accessed.

In this case, the following variables can be marked as `immutable` and initialized in the constructor to optimize gas usage:

- `token_x`
- `token_y`
- `scaling_factor_token_x`
- `scaling_factor_token_y`
- `supports_native_eth`
- `is_token_x_weth`
- `askTrie`
- `bidTrie`

The issue is classified as **low** severity because, while it does not pose a security risk, optimizing these variables can lead to more efficient contract execution and reduced gas costs.
##### Recommendation
We recommend marking these variables as `immutable` and initializing them in the constructor to achieve better gas efficiency.

---
### Example 2

**Auto Label:** Lack of immutability in state variables leads to unnecessary gas costs, compromised contract integrity, and potential runtime manipulation, undermining security and efficiency.  

**Original Text Preview:**

##### Description
The [`commission_scaling_factor`](https://github.com/longgammalabs/hanji-contracts/blob/09b6188e028650b9c1758010846080c5f8c80f8e/src/OnchainLOB.sol#L51) is set as an `immutable` variable, but it is typically a constant value (e.g., `1e6`) in many contracts. Using a constant instead of an immutable variable can optimize gas usage and improve clarity.
##### Recommendation
We recommend setting the `commission_scaling_factor` as a constant with a value of `1e6`.

---
### Example 3

**Auto Label:** Lack of immutability in state variables leads to unnecessary gas costs, compromised contract integrity, and potential runtime manipulation, undermining security and efficiency.  

**Original Text Preview:**

##### Description
The `pyth` address is stored as a regular state variable. If this address is never intended to change, making it effectively immutable could save gas on repeated reads.

The issue is classified as **low** severity because it focuses on optimization rather than a security flaw.

##### Recommendation
We recommend defining `pyth` as an `immutable` variable within the constructor if updates are unnecessary post-deployment.

---

---
