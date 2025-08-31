# Cluster -1415

**Rank:** #426  
**Count:** 10  

## Label
Unrestricted or unauthorized withdrawal of staked assets, enabling centralization, fund loss, and bypassing governance or emergency safeguards.

## Cluster Information
- **Total Findings:** 10

## Examples

### Example 1

**Auto Label:** Unrestricted or unauthorized withdrawal of staked assets, enabling centralization, fund loss, and bypassing governance or emergency safeguards.  

**Original Text Preview:**

##### Description

`ResolvStaking` and `ResolvStakingSilo` have a function `emergencyWithdrawERC20()`, which allows the admin to withdraw `RESOLV` tokens from the contract.
- https://github.com/resolv-im/resolv-contracts/blob/530d997b010b803691abab53835ad70f780ac5db/contracts/staking/ResolvStaking.sol#L212
- https://github.com/resolv-im/resolv-contracts/blob/530d997b010b803691abab53835ad70f780ac5db/contracts/staking/ResolvStakingSilo.sol#L33

The ability to withdraw `RESOLV` tokens (the main staking asset) via this function may present centralization concerns, as it could allow administrators to remove staked user funds.
<br/>
##### Recommendation
We recommend restricting the emergency withdrawal function from transferring out the main staking asset.

> **Client's Commentary:**
> Acknowledged

---

---
### Example 2

**Auto Label:** Unrestricted or unauthorized withdrawal of staked assets, enabling centralization, fund loss, and bypassing governance or emergency safeguards.  

**Original Text Preview:**

## Severity: Medium Risk

## Context
UnwindingModule.sol#L103

## Description
In the UnwindingModule, users who have begun unwinding and are part of it, still are exposed to any earned yield and losses from accrue calls. However, as accrue is not called upon withdrawals, users could front-run any accrue call that will result in a slash by calling withdraw. The remaining users in the UnwindingModule would absorb the full impact of the negative yield. This could be easily abused by users within the infiniFi protocol to be exposed to yield gains without risking any losses/future slash.

## Recommendation
Consider calling `YieldSharing.accrue` before any `UnwindingModule.withdraw` call. This way, no one in the UnwindingModule can avoid their share of a slash simply by withdrawing preemptively.

## infiniFi
Fixed in 24513bd by reverting if there are pending unaccrued losses when a user does `siUSD ! iUSD`, `liUSD ! iUSD` or `iUSD ! USDC`.

## Spearbit
Fix verified.

---
### Example 3

**Auto Label:** Unauthorized asset withdrawal without user consent or proper validation, leading to permanent fund loss or centralization of control.  

**Original Text Preview:**

## Security Assessment Report

## Severity: Low Risk

### Context
**File:** BaseManaged.sol  
**Lines:** L74-L85

### Description
All contracts in the protocol inherit `BaseManaged`, which contains the `rescueTokens()` function:

```solidity
function rescueTokens(address token, uint amountOrId, bool isNFT) external onlyOwner {
    /// The transfer is to the owner so that only full owner compromise can steal tokens
    /// and not a single rescue transaction with bad params
    if (isNFT) {
        IERC721(token).transferFrom(address(this), owner(), amountOrId);
    } else {
        // for ERC-20 must use transfer, since most implementation won't allow transferFrom
        // without approve (even from owner)
        SafeERC20.safeTransfer(IERC20(token), owner(), amountOrId);
    }
    emit TokensRescued(token, amountOrId);
}
```

However, this function allows the owner to transfer out all funds in the protocol, including any loan/provider/taker/escrow NFTs held in the contracts.

### Recommendation
Consider checking that `token` is not `cashAsset`, `underlying`, or `address(this)` in their respective contracts. This prevents the owner from transferring user funds out of the protocol.

### Collar
Acknowledged. This is unfortunately true, but the intention is to provide safety against a catastrophic bug, in which case cash or underlying is what needs to be rescued. The centralisation risk in this case is similar to an upgradeable contract, but with surface area (operational risks, and smart contract risk) reduced. We've added more explicit documentation around these concerns in commit `897a702d`.

### Spearbit
Acknowledged.

---
