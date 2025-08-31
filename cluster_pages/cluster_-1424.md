# Cluster -1424

**Rank:** #454  
**Count:** 7  

## Label
Incorrect token transfer sourcing enables unauthorized movement, reverts, and double payments due to flawed balance validation and improper recipient assignment.

## Cluster Information
- **Total Findings:** 7

## Examples

### Example 1

**Auto Label:** Incorrect token destination and state mismatches due to flawed transfer logic and inadequate balance validation, leading to fund loss or blocked claims.  

**Original Text Preview:**

## Vulnerability Report: USD0++ Token PAR Mechanism

## Context
- File: `Usd0PP.sol`
- Lines: 347-402

## Description
The USD0++ token's Parity Arbitrage Right (PAR) mechanism contains a mistake in its implementation. When activated, excess USD0 tokens are incorrectly sent to the AirdropDistribution contract instead of the intended DAO treasury, potentially making funds inaccessible.

The USD0++ token includes a PAR mechanism designed to stabilize its price relative to USD0 when it falls below parity on the open market. This mechanism can be activated when more USD0++ tokens exist on the market than USD0 tokens, and the price ratio of USD0:USD0++ is below 1. When triggered, the mechanism unwraps USD0++ for USD0 and exchanges it on the market to support the USD0++ price. Any excess USD0 received from this operation is supposed to be sent to the DAO's treasury.

However, the current implementation incorrectly forwards this excess amount to the AirdropDistribution contract. This contract is not designed to hold tokens and lacks methods for transferring them out, potentially leading to locked funds.

## Impact
The impact of this vulnerability can potentially be significant, as funds are effectively locked in the AirdropDistribution contract. Yet, as the contract is upgradeable, the impact can be lowered to medium. Furthermore, addressing this in an emergency upgrade carries its own risks and complications.

## Likelihood
The likelihood of this issue occurring is high when the PAR mechanism is activated. The PAR mechanism itself is, however, not intended to be used frequently, lowering the likelihood of the vulnerability occurring to medium.

## Recommendation
To address this issue, modify the `triggerPARMechanismCurvepool` function in the Usd0PP contract to transfer the USD0 tokens to the correct address:

```solidity
usd0.safeTransfer(
- $.registryContract.getContract(CONTRACT_AIRDROP_DISTRIBUTION), gainedUSD0AmountPAR
+ $.registryContract.getContract(CONTRACT_TREASURY), gainedUSD0AmountPAR
);
```

## Usual
Fixed in commit `ae6ea94e`.

## Cantina Managed
Fixed.

---
### Example 2

**Auto Label:** Incorrect token destination and state mismatches due to flawed transfer logic and inadequate balance validation, leading to fund loss or blocked claims.  

**Original Text Preview:**

## Severity

**Impact:** Low

**Likelihood:** High

## Description

When users creates DAO they specify the `claimBalance` and later they can increase it. The issue is that when `hasAllowanceMechanism` is set there's no way to increase `claimBalance` because `depositTokens()` won't allow it. This would cause constant `claimBalance` that admin set during the DAO creation and it won't be possible to increase airdrop total amount.

## Recommendations

Allow admins to increase `claimBalance` when `hasAllowanceMechanism` is set.

---
### Example 3

**Auto Label:** Incorrect token destination and state mismatches due to flawed transfer logic and inadequate balance validation, leading to fund loss or blocked claims.  

**Original Text Preview:**

##### Description
The agreement creation precedes the token transfer. If a fee on transfer tokens is used, then the amount of tokens transferred may be reduced (due to transfer fees) and become less than the amount specified in the agreement for claiming.

This issue is labeled as `medium` since the resulted inconcistency can block the `claim` function until the `balance` of the timelock surpasses the quantity of tokens noted in the agreement.

Related code - agreement creation: https://github.com/dforce-network/LendingContracts/blob/6f3a7b6946d8226b38e7f0d67a50e68a28fe5165/contracts/DefaultTimeLock.sol#L47
##### Recommendation
We recommend reworking the `TimeLock` architecture to pull assets by `transferFrom` in the `createAgreement`. Additionally, it will help addressing previous finding.

---
