# Cluster -1010

**Rank:** #276  
**Count:** 29  

## Label
Unused debugging logs in production contracts expose sensitive internal state and execution details, increasing attack surface through information leakage and unnecessary gas costs.

## Cluster Information
- **Total Findings:** 29

## Examples

### Example 1

**Auto Label:** Unused debugging logs in production contracts expose sensitive internal state and execution details, increasing attack surface through information leakage and unnecessary gas costs.  

**Original Text Preview:**

##### Description

Most contracts in scope include the following import:

```
import "hardhat/console.sol";
```

  

The `console.sol` library is a debugging utility from Foundry, typically used for logging during development and testing. Including it in production contracts increases gas costs and can unintentionally expose sensitive information or internal logic.

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:N/S:C (0.0)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:N/S:C)

##### Recommendation

Remove the `hardhat/console.sol` import and any associated debugging statements from production code before deployment.

##### Remediation

**PENDING:** The **B14G team** indicated that they will be addressing this finding in the future.

---
### Example 2

**Auto Label:** Unused debugging logs in production contracts expose sensitive internal state and execution details, increasing attack surface through information leakage and unnecessary gas costs.  

**Original Text Preview:**

##### Description

Most contracts in scope include the following import:

```
import "hardhat/console.sol";
```

  

The `console.sol` library is a debugging utility from Foundry, typically used for logging during development and testing. Including it in production contracts increases gas costs and can unintentionally expose sensitive information or internal logic.

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:N/S:C (0.0)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:N/S:C)

##### Recommendation

Remove the `hardhat/console.sol` import and any associated debugging statements from production code before deployment.

##### Remediation

**PENDING:** The **B14G team** indicated that they will be addressing this finding in the future.

---
### Example 3

**Auto Label:** Unused debugging logs in production contracts expose sensitive internal state and execution details, increasing attack surface through information leakage and unnecessary gas costs.  

**Original Text Preview:**

## Code Review Summary

## Context
- `LibPrice.sol#L43-L51`
- `LibPrice.sol#L88`
- `LibPrice.sol#L103`
- `LibPrice.sol#L109-L115`

## Description
There are multiple `console.log()` statements in `LibPrice.sol`. It’s a best practice to avoid using them in production-ready code.

## Recommendation
Consider removing `console.log()` statements before deploying contracts.

## Action Taken
- **PintoFarm**: Addressed in PR 16.
- **Cantina Managed**: Fix verified.

---
