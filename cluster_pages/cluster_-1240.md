# Cluster -1240

**Rank:** #57  
**Count:** 300  

## Label
Failure to wrap residual or excess native ETH into WETH before returning it to users, leading to asset loss and incorrect trade reporting due to improper handling of ETH in swap and deposit flows.

## Cluster Information
- **Total Findings:** 300

## Examples

### Example 1

**Auto Label:** Failure to properly handle or refund incoming ETH leads to permanent fund loss or trapped assets due to missing value validation, receipt mechanisms, or refund paths.  

**Original Text Preview:**

When the user calls `requestUnstake()`, `msg.value` is required as part of the payment for serviceFee. However, if the user sends more `msg.value` than required, then the excess value will be stuck in the contract.

```
    // Ensure that the fee amount is sufficient
    require(msg.value >= serviceFee, "Fee amount is not sufficient");
```

Recommend having a strict equality check instead.

---
### Example 2

**Auto Label:** Failure to properly handle or refund incoming ETH leads to permanent fund loss or trapped assets due to missing value validation, receipt mechanisms, or refund paths.  

**Original Text Preview:**

##### Description
In `LimitOrderManager.algebraMintCallback()` the debts to the pool are paid in order: first `token0`, then `token1`.

When `token0` is not `wNativeToken` but `token1` is, and the user supplies ETH with the call:

1. `_pay(token0, …)` executes the ERC-20 branch, transferring `token0` from the user.
2. The final `if (address(this).balance > 0)` line refunds all ETH held by the contract back to the payer, even though it will be needed momentarily.
3. `_pay(token1, …)` now finds the contract’s ETH balance at zero, cannot wrap ETH into `WNativeToken`, and reverts.

The order fails despite the user providing sufficient ETH, interrupting liquidity provision and forcing a retry. 

This finding has been classified as **Medium** severity because legitimate actions are being blocked.
<br/>
##### Recommendation
We recommend refunding the surplus ETH after both payments are processed.

> **Client's Commentary:**
> Fixed. Commit: https://github.com/cryptoalgebra/plugins-monorepo/commit/417edc77ad50789fb16fbab50d697b29e967cca6

---

---
### Example 3

**Auto Label:** Failure to properly handle or refund incoming ETH leads to permanent fund loss or trapped assets due to missing value validation, receipt mechanisms, or refund paths.  

**Original Text Preview:**

##### Description
When users call `LimitOrderManager.place()` with excess `msg.value`, the extra ETH remains in the `LimitOrderManager` balance instead of being forwarded or refunded. When `algebraMintCallback()` is called, that balance is wrapped into `WNativeToken` and sent to the pool, but any surplus never returns to the user. Conversely, if `msg.value` is lower than the amount of `WNativeToken` required for mint and the user previously gave `WNativeToken` allowance to `LimitOrderManager`, `LimitOrderPayments._pay()` pulls ERC-20 tokens in addition to consuming the partial ETH, resulting in a double charge.

Because this directly risks user funds by locking or over-charging assets, it is classified as **Medium** severity.
<br/>
##### Recommendation
We recommend including the original `msg.value` in `MintCallbackData`, and then calculating how much ETH the `_pay` operation actually consumed and refunding any remainder back to the caller.

> **Client's Commentary:**
> Commit: https://github.com/cryptoalgebra/plugins-monorepo/commit/16f2df49c11cc3f48374a0cbd9156927b830f64d. Since the `LimitOrderManager` contract is not expected to hold any native currency, we can refund the user the entire remaining contract balance. 

---

---
