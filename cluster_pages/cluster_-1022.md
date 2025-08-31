# Cluster -1022

**Rank:** #322  
**Count:** 21  

## Label
Inaccurate reserve or share calculations due to flawed arithmetic or missing inputs, leading to premature unwinds, inflated user shares, or broken pool mechanics and potential asset loss.

## Cluster Information
- **Total Findings:** 21

## Examples

### Example 1

**Auto Label:** Reserve overstatement due to flawed accounting of executed or returned shares, leading to inflated liquidity and potential protocol insolvency.  

**Original Text Preview:**

##### Description
This issue has been identified within the `_claimOrder` function of the `ProxyLOB` contract. 

When `isAsk = false`, the current code subtracts `(totalClaimedShares + passiveFee) * scalingFactorTokenY` from the reserves without multiplying `totalClaimedShares` by `price`. Consequently, the actual value of the claimed shares is not correctly accounted for, resulting in a higher-than-accurate reserves value.

The issue is classified as **critical** severity because it leads to a miscalculation of the claimed token’s worth, which inflates the protocol’s reported balances.

##### Recommendation
We recommend using `(totalClaimedShares * price + passiveFee) * scalingFactorTokenY` for the subtraction. This ensures the claimed amount is converted into its proper value before being removed from the reserves.


***

---
### Example 2

**Auto Label:** Reserve overstatement due to flawed accounting of executed or returned shares, leading to inflated liquidity and potential protocol insolvency.  

**Original Text Preview:**

##### Description
This issue has been identified within the `_claimOrder` function of the `ProxyLOB` contract. 

When placing a bid (`isAsk = false`), the contract transfers `quantity * price + passivePayoutCommission` into the LOB and records the same total in `lobReservesByTokenId`. However, if the order is partially executed, and `claimOrder` is invoked with `only_claim = false`, the unfilled portion of the fee is returned to the LPManager but is never subtracted from `lobReservesByTokenId`. Consequently, `_getReserves` continues to count fees that no longer remain, causing an inflated reserve calculation.

The issue is classified as **critical** severity because overstating reserves can lead to protocol insolvency if the system operates on incorrect balance assumptions.

##### Recommendation
We recommend updating `_claimOrder` to remove the unclaimed fee portion from `lobReservesByTokenId` once it is returned to the LPManager. This prevents double-counting in reserve calculations. For instance, change:
```solidity!
uint256 executedValue = executedShares * price;
uint256 passiveFee = executedValue.mulWadUp(passiveCommissionRate);
```
to:
```solidity!
uint256 returnedValue = totalClaimedShares * price;
uint256 passiveFee = returnedValue.mulWadUp(passiveCommissionRate);
```

---

---
### Example 3

**Auto Label:** Reserve overstatement due to flawed accounting of executed or returned shares, leading to inflated liquidity and potential protocol insolvency.  

**Original Text Preview:**

##### Description
This issue has been identified within the `_placeOrder` function of the `ProxyLOB` contract. 

When `marketOnly` is set to `true` the order is not included into `IOnchainTrie` within the `IOnchainLOB.placeOrder`, but the code still calculates `remainShares` as `quantity - executedShares`, instead of correctly setting it to zero. As a result, these non-existent leftover shares are added to `lobReservesByTokenId` and factored into `_getReserves`, ultimately inflating the protocol’s reported total balance. Since `_getReserves` then shows a value higher than what truly exists, this discrepancy can lead to insolvency of the protocol.

The issue is classified as **critical** severity because overstated reserves severely undermine the protocol’s financial stability, risking potential losses for liquidity providers.

##### Recommendation
We recommend setting `remainShares` to zero when `marketOnly` is `true`, ensuring no residual shares are mistakenly recorded. This accurately reflects that market orders have no remaining portion and preserves correct reserve accounting.

---

---
