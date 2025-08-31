# Cluster -1461

**Rank:** #387  
**Count:** 14  

## Label
Failure to validate real-time price movements or transaction logic leads to exploitable price manipulation, missed liquidations, and risk-free profit extraction through timing and market abuse.

## Cluster Information
- **Total Findings:** 14

## Examples

### Example 1

**Auto Label:** Failure to validate real-time price movements or transaction logic leads to exploitable price manipulation, missed liquidations, and risk-free profit extraction through timing and market abuse.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

The protocol's signature validation mechanism enables option mispricing through period manipulation and stale signatures:

1. **Period and Maturity Mismatch**

```solidity
function _hash(...) internal pure returns (bytes32) {
@>  return keccak256(abi.encode(iv, uAsset, oType, maturityDate, relativeStrike));
}

function iv(uint48 period, OptionType oType, uint256 strikePrice) external view returns (uint256 value) {
@>  value = _ivValues[period][oType][strikePrice]; // IV is queried based on `period`, `oType` and `strikePrice`
}
```

2. **Missing Signature Deadline**

```solidity
function _recoverSigner(...) internal view returns (address) {
@>  address recoveredSigner = _recoverSigner(sig, iv, strategy, period._computeMaturityDate(), relativeStrike);
}
```

This enables two primary attack vectors:

1. **Period Manipulation**

- User requests signature for a 14-day period on April 3 (maturity: April 17 23:59).
- On April 10, the user executes the option with a 7-day period (maturity: April 17 23:59, results in the same maturity).
- The process validates this position with IV intended for a 14-day period, mispricing the option.

2. **Stale Strike Price**

- User requests signature for a 10% relative strike at a spot price 3500 USD(strike = 3850 USD).
- User waits until the market moves to some spot price, ie 4200 USD (new strike = 4620 USD).
- User executes the option with a stale signature, still using the IV for strike 3850 USD.
- The protocol applies an outdated IV, mispricing the option.

## Recommendation

Include all critical parameters and enforce signature expiration validation:

```diff
function _hash(...) internal pure returns (bytes32) {
    return keccak256(abi.encode(
        iv,
        uAsset,
        oType,
+       period,
        maturityDate,
        relativeStrike,
+       deadline
    ));
}
```

---
### Example 2

**Auto Label:** Poor validation and ambiguous logic in price-related functions lead to incorrect order execution, unreliable pricing, and potential reentrancy or state manipulation.  

**Original Text Preview:**

##### Description
This issue has been identified in the current design, which relies on `validatePriceMovement` for both updating and validating price movements with multiple parameters. This approach can lead to confusion about the purpose of each parameter and makes the contract logic harder to audit and maintain. 

The issue is classified as **low** severity because it does not introduce a direct vulnerability but negatively affects code clarity and maintainability.

##### Recommendation
We recommend separating the logic into two dedicated functions:
1. **`resetPrice()`**: Responsible only for fetching and storing the default price.
2. **`validatePriceMovement()`**: Called afterward to verify that price movements remain within expected boundaries.

This separation of functionality will make the code more intuitive, minimize confusion about parameters, and improve long-term maintainability.

---

---
### Example 3

**Auto Label:** Poor validation and ambiguous logic in price-related functions lead to incorrect order execution, unreliable pricing, and potential reentrancy or state manipulation.  

**Original Text Preview:**

##### Description
This issue has been identified in the current design, which relies on `validatePriceMovement` for both updating and validating price movements with multiple parameters. This approach can lead to confusion about the purpose of each parameter and makes the contract logic harder to audit and maintain. 

The issue is classified as **low** severity because it does not introduce a direct vulnerability but negatively affects code clarity and maintainability.

##### Recommendation
We recommend separating the logic into two dedicated functions:
1. **`resetPrice()`**: Responsible only for fetching and storing the default price.
2. **`validatePriceMovement()`**: Called afterward to verify that price movements remain within expected boundaries.

This separation of functionality will make the code more intuitive, minimize confusion about parameters, and improve long-term maintainability.

---

---
