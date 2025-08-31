# Cluster -1289

**Rank:** #365  
**Count:** 16  

## Label
Inconsistent fee handling across order types or flows enables users to bypass intended fees, leading to revenue loss and protocol misalignment through asymmetric or unenforced fee application.

## Cluster Information
- **Total Findings:** 16

## Examples

### Example 1

**Auto Label:** Inconsistent or flawed fee handling mechanisms that lead to unexpected user experiences, fee loss, or denial-of-service, often due to poor logic, lack of exemptions, or improper validation.  

**Original Text Preview:**

## Summary

When the total fee recipient shares reach the maximum limit, reducing a recipient’s share is blocked due to a validation check. This prevents owners from updating feeBps for existing recipients.

## Vulnerability Details

The `configureFeeRecipient` allows owner to add , remove or update shares of recipient’s . when we calls this function it first validate that the newShare value will not exceeds the max limit of allows  recipients’s shares.

```solidity
/home/aman/Desktop/audits/2025-01-zaros-part-2/src/market-making/branches/MarketMakingEngineConfigurationBranch.sol:613
613:     function configureFeeRecipient(address feeRecipient, uint256 share) external onlyOwner {
614:         // revert if protocolFeeRecipient is set to zero
615:         if (feeRecipient == address(0)) revert Errors.ZeroInput("feeRecipient");
616: 
617:         // load market making engine configuration data from storage
618:         MarketMakingEngineConfiguration.Data storage marketMakingEngineConfiguration =
619:             MarketMakingEngineConfiguration.load();
620: 
621:         // check if share is greater than zero to verify the total will not exceed the maximum shares
622:         if (share > 0) {
623:             UD60x18 totalFeeRecipientsSharesX18 = ud60x18(marketMakingEngineConfiguration.totalFeeRecipientsShares);
624: 
625:             if (
626:                 totalFeeRecipientsSharesX18.add(ud60x18(share)).gt(
627:                     ud60x18(Constants.MAX_CONFIGURABLE_PROTOCOL_FEE_SHARES)
628:                 )
629:             ) {
630:                 revert Errors.FeeRecipientShareExceedsLimit();
631:             }
632:         }
633: 
634:         (, uint256 oldFeeRecipientShares) = marketMakingEngineConfiguration.protocolFeeRecipients.tryGet(feeRecipient);
635: 
636:         // update protocol total fee recipients shares value
637:         if (oldFeeRecipientShares > 0) {
638:             if (oldFeeRecipientShares > share) {
639:                 marketMakingEngineConfiguration.totalFeeRecipientsShares -=
640:                     (oldFeeRecipientShares - share).toUint128();
641:             } else {
642:                 marketMakingEngineConfiguration.totalFeeRecipientsShares +=
643:                     (share - oldFeeRecipientShares).toUint128();
644:             }
645:         } else {
646:             marketMakingEngineConfiguration.totalFeeRecipientsShares += share.toUint128();
647:         }
648: 
649:         // update protocol fee recipient
650:         marketMakingEngineConfiguration.protocolFeeRecipients.set(feeRecipient, share);
651: 
652:         // emit event LogConfigureFeeRecipient
653:         emit LogConfigureFeeRecipient(feeRecipient, share);
654:     }
```

The above code will not work as intended , as in case if the `totalFeeRecipientsSharesX18=MAX_CONFIGURABLE_PROTOCOL_FEE_SHARES` and owner wants to decrease share of specfic recipient’s it will always revert  due to check before removal of shares. The following POC will demonstrates it.

## POC

```solidity
/test/integration/market-making/market-making-engine-configuration-branch/configureFeeRecipient/configureFeeRecipieint.t.sol:39
39:     function test_totalFeeRecipientsShare_is_max_poc() external { // @audit POC
40:         address user1 = address(0x1234);
41:         address user2 = address(0x5678);
42:         marketMakingEngine.configureFeeRecipient(user1, 0.8e18); // set user1 share to 0.8
43:         marketMakingEngine.configureFeeRecipient(user2, 0.1e18); // set user2 share to 0.1
44:         // shares are already at max limit i.e 0.9e18
45:         // will not allow to update user1 share after words
46:         marketMakingEngine.configureFeeRecipient(user1, 0.7e18); // update user1 share to 0.7 
47:     }
```

run the Test with Command : `forge test --mt test_totalFeeRecipientsShare_is_max_poc`

## Impact

Owners cannot reduce a recipient’s feeBps when the total fee recipients’ shares are at the maximum limit.

## Tools Used

Manual Review

## Recommendations

Modify the configureFeeRecipient function to allow fee share reductions even when the total shares are at the max limit. Proposed Fix:

```diff
diff --git a/src/market-making/branches/MarketMakingEngineConfigurationBranch.sol b/src/market-making/branches/MarketMakingEngineConfigurationBranch.sol
index 6fcb388..c374564 100644
--- a/src/market-making/branches/MarketMakingEngineConfigurationBranch.sol
+++ b/src/market-making/branches/MarketMakingEngineConfigurationBranch.sol
@@ -621,17 +621,7 @@ contract MarketMakingEngineConfigurationBranch is OwnableUpgradeable {
 
         // check if share is greater than zero to verify the total will not exceed the maximum shares
         // @audit : what if share is already on 0.9e18 limit , and we want to decrease share of a user how can we do that ?
-        if (share > 0) {
-            UD60x18 totalFeeRecipientsSharesX18 = ud60x18(marketMakingEngineConfiguration.totalFeeRecipientsShares);
-
-            if (
-                totalFeeRecipientsSharesX18.add(ud60x18(share)).gt(
-                    ud60x18(Constants.MAX_CONFIGURABLE_PROTOCOL_FEE_SHARES)
-                )
-            ) {
-                revert Errors.FeeRecipientShareExceedsLimit();
-            }
-        }
+        
 
         (, uint256 oldFeeRecipientShares) = marketMakingEngineConfiguration.protocolFeeRecipients.tryGet(feeRecipient);
 
@@ -649,7 +639,17 @@ contract MarketMakingEngineConfigurationBranch is OwnableUpgradeable {
         } else {
             marketMakingEngineConfiguration.totalFeeRecipientsShares += share.toUint128();
         }
+        if (share > 0) {
+            UD60x18 totalFeeRecipientsSharesX18 = ud60x18(marketMakingEngineConfiguration.totalFeeRecipientsShares);
 
+            if (
+                totalFeeRecipientsSharesX18.gt(
+                    ud60x18(Constants.MAX_CONFIGURABLE_PROTOCOL_FEE_SHARES)
+                )
+            ) {
+                revert Errors.FeeRecipientShareExceedsLimit();
+            }
+        }
         // update protocol fee recipient
         marketMakingEngineConfiguration.protocolFeeRecipients.set(feeRecipient, share);

```

---
### Example 2

**Auto Label:** Inconsistent or flawed fee handling mechanisms that lead to unexpected user experiences, fee loss, or denial-of-service, often due to poor logic, lack of exemptions, or improper validation.  

**Original Text Preview:**

## SimpleSwapFacility.sol - Fee Mechanism Overview

## Context
**Location:** SimpleSwapFacility.sol#L181

## Description
In `SimpleSwapFacility.sol`, the `feeRecipient` collects fees in `debtToken` whenever users exchange their collateral for debt, or vice versa. These fees accumulate over time and can be collected through the `collectFees()` function, where the `feeRecipient` receives debt tokens. 

To convert these debt tokens back to collateral tokens, they must use the `swapExactDebtForCollateral()` function, which incurs a swap-out fee rate. As a result, a portion of the fees earned will be recycled back into the fee accumulator. This creates a cycle of fee-on-fee charges whenever they attempt to convert any part of their tokens.

For instance, if both swap fees are set at 100 basis points (1%), the fee recipient would incur a loss of 1% of their earned fees when trying to convert them to collateral. These deducted fees would then be added back to the total accumulated fees, ultimately credited to the fee recipient again, creating an infinite loop.

## Recommendation
Consider creating a privileged swap function for the `feeRecipient` to swap without swap-out fees.

## Bitcorn
Acknowledging and keeping this in mind as a future upgrade option. Original behavior will be retained.

If there are fees and the `feeRecipient` wants to redeem the underlying, the following approach will approximate the behavior described. Given that the `feeRecipient` is the same entity as the administrative role holder, you can temporarily allow redemption without fees by sandwiching the `collectFees()` call (which is permissioned) between calls that set fees to zero and then set them to the original value after the call.

This can be atomically executed by a multisig or other smart contract wallet, or a zap could be made to execute it in a way that practically has the same result as the proposed solution at the expense of more gas usage.

## Cantina Managed
Acknowledged.

---
### Example 3

**Auto Label:** Inconsistent or flawed fee handling mechanisms that lead to unexpected user experiences, fee loss, or denial-of-service, often due to poor logic, lack of exemptions, or improper validation.  

**Original Text Preview:**

## Context: SwapFacility.sol#L293

## Description
If `_feeRate` were to be 0, and this code path is reachable, the contract would revert on any code paths that attempt this fee calculation, thereby allowing owners to partially pause the affected functions. The contract in its current state is not affected thanks to the preconditional checks prior to its two calls that `feeRate` is greater than 0, but as it is upgradeable, future code changes may miss adding that necessary step.

## Recommendation
It is recommended to have the non-zero fee rate check within the function itself, and return a fee of 0 early in a null case, skipping the calculation. This will additionally remove the need for duplicated checks of this with preconditions in the various swap code paths.

## Bitcorn
Mitigated by adding early return with `feeRate = 0` in commit bef838b2.

## Cantina Managed
Verified fix.

---
