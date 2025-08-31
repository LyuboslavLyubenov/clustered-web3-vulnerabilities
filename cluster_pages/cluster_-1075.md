# Cluster -1075

**Rank:** #192  
**Count:** 59  

## Label
Lack of proper access control and validation enables unauthorized or malicious actors to execute critical operations, bypassing intended safeguards through insufficient checks on callers, addresses, or contract interfaces.

## Cluster Information
- **Total Findings:** 59

## Examples

### Example 1

**Auto Label:** Unauthorized function execution via insufficient access control and signature validation, enabling attackers to exploit or drain funds through arbitrary or impersonated calls.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** High

## Description

The `OmoVault.sol` contract implements some principles of the ERC7540 standard, Asynchronous redeem shares for assets.
However, users are still able to withdraw directly using `ERC4626.sol#withdraw()`

Also, `_validateSignature()` function in `DynamicAccount.sol` contract accepts the user wallet as a signer, through the Entry Point.
So, the user has the same privileges as the agent.

Malicious users can call `withdraw()` from the vault directly to receive their funds.
Next, he will transfer all the funds and the UniswapV3 positions out of their Dynamic Account

As a result, he will duple his first deposit in the vault.

## Recommendations

1- Override `withdraw()` function in `OmoVault.sol` and make it revert.
2- In `_validateSignature()` you need to check what function can be called if the signer is the owner

---
### Example 2

**Auto Label:** Inadequate access control enables unauthorized modifications or operations, leading to denial of service or unauthorized access through flawed authorization checks and missing caller validations.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-10-ethos-network-judging/issues/23 

## Found by 
056Security, 0x37, 0xBhumii, 0xMosh, 0xbrivan, Bozho, DigiSafe, Dliteofficial, IvanFitro, John44, KlosMitSoss, LeFy, PNS, TessKimy, dany.armstrong90, debugging3, dobrevaleri, durov, heeze, justAWanderKid, newspacexyz, onthehunt, s0x0mtee, sammy, shaflow01, t.aksoy, y4y
### Summary

A missing compromise check in `verifiedProfileIdForAddress` will cause unauthorized access for affected contracts as compromised addresses can bypass security measures and perform malicious actions.
(e.g: Attacker can steal user's private key, so address is compromised)

### Root Cause

In `EthosProfile.sol`, the `verifiedProfileIdForAddress` function is missing a check to ensure `_address` is not compromised, allowing compromised addresses to interact with other contracts without restriction.
https://github.com/sherlock-audit/2024-10-ethos-network/blob/979e352d7bcdba3d0665f11c0320041ce28d1b89/ethos/packages/contracts/contracts/EthosProfile.sol#L568-L574

The `verifiedProfileIdForAddress` function used in many contracts.

Here: 
https://github.com/sherlock-audit/2024-10-ethos-network/blob/979e352d7bcdba3d0665f11c0320041ce28d1b89/ethos/packages/contracts/contracts/EthosAttestation.sol#L228
https://github.com/sherlock-audit/2024-10-ethos-network/blob/979e352d7bcdba3d0665f11c0320041ce28d1b89/ethos/packages/contracts/contracts/EthosDiscussion.sol#L111-L113
https://github.com/sherlock-audit/2024-10-ethos-network/blob/979e352d7bcdba3d0665f11c0320041ce28d1b89/ethos/packages/contracts/contracts/EthosDiscussion.sol#L158-L160
...

In this project, there are many issues about this compromised address.
In almost contracts, it doesn't check `msg.sender` is compromised address.
Address is already unregistered from profile by `deleteAddressAtIndex` function, but it is still used in many functions.

### Internal pre-conditions

User needs to call `deleteAddressAtIndex` to set `isAddressCompromised` to be true for the target address.

### External pre-conditions

_No response_

### Attack Path

1. Attack steal user's private key.
2. User detected it is compromised and calls `deleteAddressAtIndex` for marking `isAddressCompromised` as true for the target address.
3. Attacker can calls `addReview` in `EthorsReview` contract by compromised address. (private key is stolen so attacker can do this operation)
It calls `ethosProfile.verifiedProfileIdForAddress(msg.sender);` msg.sender is compromised but it doesn't revert.

### Impact

The **protocol** suffers a potential security breach as **compromised addresses** can bypass verification and execute unauthorized actions in dependent contracts, potentially leading to **manipulation of contract functionality**. The attacker gains access to otherwise restricted operations without proper authorization.

### PoC

_No response_

### Mitigation

Add modifier `checkIfCompromised` and use `checkIsAddressCompromised` function.



## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/trust-ethos/ethos/pull/1836

---
### Example 3

**Auto Label:** Inadequate access control enables unauthorized modifications or operations, leading to denial of service or unauthorized access through flawed authorization checks and missing caller validations.  

**Original Text Preview:**

##### Description
`createDirectGrant()`, `createBasicGrant()`, `createAdvancedGrant()` can use the same internal method to decrease the size of the contract. https://github.com/MetaLex-Tech/borg-core/blob/9074503d37cfa1d777ef16f6c88b84c98b4f54eb/src/implants/optimisticGrantImplant.sol#L101-L255

##### Recommendation
We recommend making improvements from the description.

---
