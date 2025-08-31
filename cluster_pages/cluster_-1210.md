# Cluster -1210

**Rank:** #382  
**Count:** 14  

## Label
**Inaccurate or missing fee calculations due to flawed cost modeling, improper validation, or lack of real-time on-chain visibility, leading to financial loss, incorrect transactions, or user risk.**

## Cluster Information
- **Total Findings:** 14

## Examples

### Example 1

**Auto Label:** **Inaccurate or missing fee calculations due to flawed cost modeling, improper validation, or lack of real-time on-chain visibility, leading to financial loss, incorrect transactions, or user risk.**  

**Original Text Preview:**

## Severity

Informational

## Description

Currently, relayers calling the withdraw function lack an efficient way to determine the
exact fee they will receive for processing a withdrawal. This uncertainty may discourage participation
or lead to inefficient relaying strategies.

## Recommendation

To address this, an on-chain view function should be implemented to allow re-
layers to see the exact fee amount they will receive before executing a withdrawal transaction.

## Team Response

Fixed.

---
### Example 2

**Auto Label:** **Inaccurate or missing fee calculations due to flawed cost modeling, improper validation, or lack of real-time on-chain visibility, leading to financial loss, incorrect transactions, or user risk.**  

**Original Text Preview:**

Buying tokens, or deploying DAOs incurs many variable fees, depending on KYC, cross-chain transactions, and fee percentages.

No view function allows users to easily request how much it will cost them to pay accordingly.

This can be quite challenging for a UI without an easy way to access this data. Consider adding such a function.

---
### Example 3

**Auto Label:** **Inaccurate or missing fee calculations due to flawed cost modeling, improper validation, or lack of real-time on-chain visibility, leading to financial loss, incorrect transactions, or user risk.**  

**Original Text Preview:**

## Vulnerability Report

## Severity
**Low Risk**

## Context
`FeeOracleV1.sol#L68-L72`

## Description
`execGasPrice` and `dataGasPrice` might be 0 due to division errors.

## Recommendation
Instead of checking whether `X.{gasPrice, toNativeRate}` are non-zero, it might be best to check whether `execGasPrice` and `dataGasPrice` are non-zero:

```solidity
function feeFor(uint64 destChainId, bytes calldata data, uint64 gasLimit) external view returns (uint256) { 
    IFeeOracleV1.ChainFeeParams storage execP = _feeParams[destChainId];
    IFeeOracleV1.ChainFeeParams storage dataP = _feeParams[execP.postsTo];
    
    uint256 execGasPrice = execP.gasPrice * execP.toNativeRate / CONVERSION_RATE_DENOM;
    uint256 dataGasPrice = dataP.gasPrice * dataP.toNativeRate / CONVERSION_RATE_DENOM;
    
    require(execGasPrice > 0, "FeeOracleV1: no fee params");
    require(dataGasPrice > 0, "FeeOracleV1: no fee params");
    // ...
}
```

## Omni
Fixed in commit `ae390993` by implementing the auditor's recommendation.

## Spearbit
Verified.

---
