# Cluster -1026

**Rank:** #418  
**Count:** 10  

## Label
Inconsistent liquidity or asset threshold validations lead to incorrect edge-case handling, risking underpayment, failed withdrawals, or violated invariants due to unit mismatches or unhandled minimum thresholds.

## Cluster Information
- **Total Findings:** 10

## Examples

### Example 1

**Auto Label:** Failure to properly validate or enforce minimum balance thresholds during token redemption, leading to incorrect liquidity reporting, partial or failed redemptions, and potential fund locking or reverts.  

**Original Text Preview:**

## Severity: Low Risk

## Context
- `BuidlUSDCSource.sol#L168`
- `BuidlUSDCSource.sol#L196-L198`

## Description
The `BuidlUSDCSource` contract reports the amount of USDC tokens it can provide through its `availableToWithdraw` function, and redeems the BUIDL tokens it holds in order to get this USDC amount. Due to how the Ondo token router is set up, the amount reported by `availableToWithdraw` must always be available for withdrawal from the token contract. However, this can get violated if the contract is holding less BUIDL tokens than the minimum redeem limit. The `availableToWithdraw` simply reports the current BUIDL balance of the contract, expecting it all to be available for withdrawal.

```solidity
uint256 availableRedeemerLiquidity = ISettlement(buidlRedeemer.settlement()).availableLiquidity();
uint256 availableBuidlRedemptionTotal = currentBuidlBalance > availableRedeemerLiquidity
    ? availableRedeemerLiquidity
    : currentBuidlBalance;
return currentUSDCBalance + availableBuidlRedemptionTotal;
```

But if the contract holds less than the minimum BUIDL redemption amount, none of its BUIDL tokens are actually redeemable. In that case, the contract will be over-reporting the amount of USDC liquidity it can provide and will thus revert if the reported amount is attempted to be withdrawn from it.

## Proof of Concept
Assume the contract has 0 USDC and 190k BUIDL tokens. Assume the minimum redemption limit is 100k BUIDL. When a user calls to withdraw 90k USDC tokens, the router calls this contract to redeem 100k BUIDL tokens for 100k USDC tokens. After paying out the user, the contract is left with 10k USDC and 90k BUIDL tokens.

Now, the `availableToWithdraw` reports that it can provide 10k + 90k = 100k of liquidity. But if this 100k liquidity is to be withdrawn, the contract needs to redeem the 90k BUIDL balance, which is impossible since it falls below the minimum redemption limit. Thus, the contract can only provide up to 10k USDC.

## Recommendation
In the `availableToWithdraw` function, add the following:

```solidity
currentBuidlBalance = currentBuidlBalance >= minBUIDLRedeemAmount ? currentBuidlBalance : 0;
```

This will make sure the BUIDL balance is only considered if it is actually redeemable.

## Ondo Finance
Fixed in commit `fa8c22c3`.

## Spearbit
Fix verified.

---
### Example 2

**Auto Label:** Failure to properly validate or enforce minimum balance thresholds during token redemption, leading to incorrect liquidity reporting, partial or failed redemptions, and potential fund locking or reverts.  

**Original Text Preview:**

## Severity: Low Risk

## Context 
PSMSource.sol#L187

## Description
The USDS peg stabilization module can be used to swap USDS for USDC tokens and vice versa at 1:1 ratios. The current USDS PSM actually just uses the older DAI PSM module underneath it. Thus, during a USDC to USDS swap, the following steps take place:

1. USDC is transferred from the user to the USDS PSM contract at `0xA188EEC8F81263234dA3622A406892F3D630f98c`.
2. The USDS PSM contract uses the USDC to buy DAI from the DAI PSM contract at `0xf6e72Db5454dd049d0788e411b06CfAF16853042`.
3. The DAI is then migrated to USDS.

Thus, this flow only works as long as there is enough DAI in the DAI PSM module. The `availableToWithdraw` in this contract, however, reports the amount of USDS available as the USDC balance of the contract itself. This can be incorrect in case not enough DAI is available in the PSM contracts. The `availableToWithdraw` reports a value higher than is actually available, which can lead to reverts in the Ondo token router contract since it always expects the reported available balance to be withdrawable.

## Recommendation
When reporting the available amount of USDS tokens, the amount of DAI available in the DAI PSM contract needs to be considered.

## Ondo Finance
Fixed in commit `89a5f6ad`.

## Spearbit
Fix verified.

---
### Example 3

**Auto Label:** Failure to properly validate or enforce minimum balance thresholds during token redemption, leading to incorrect liquidity reporting, partial or failed redemptions, and potential fund locking or reverts.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-08-midas-minter-redeemer-judging/issues/100 

## Found by 
0xadrii, 0xpiken, Drynooo, Hunter, Ironsidesec, NoOneWinner, WildSniper, eeyore, pashap9990, pkqs90, rzizah

## Summary

RedemptionVaultWIthBUIDL does not redeem full balance if BUIDL balance is less than 250k post transaction.

## Vulnerability Detail

According to the [specs](https://ludicrous-rate-748.notion.site/8060186191934380800b669406f4d83c?v=35634cda6b084e2191b83f295433efdf&p=927832e82a874221996c1edcc1d94b17&pm=s), there should be a feature that when redeeming BUIDL tokens, it should redeem full balance if the remaining BUIDL tokens is less than 250k. However, no such feature is implemented.

> Redeem the full BUIDL balance in the smartcontract if the BUIDL balance will be less than 250k post transaction (as 250k is the minimum).
Make this 250k threshold a parameter that can be adjusted by the admin

Contest readme states that "Please note that discrepancies between the spec and the code can be reported as issues", thus reporting this as a medium severity issue.

Also, note that this feature is required because BUIDL token has a minimum redemption limit of 250k (according to https://www.steakhouse.financial/projects/blackrock-buidl). Thus lack of this feature may result in lock of BUIDL tokens within the RedemptionVaultWIthBUIDL contract.

> However, shares cannot be sold back unless their total value is at least $250,000, or if an exception is granted. 

```solidity
    function _checkAndRedeemBUIDL(address tokenOut, uint256 amountTokenOut)
        internal
    {
        uint256 contractBalanceTokenOut = IERC20(tokenOut).balanceOf(
            address(this)
        );
        if (contractBalanceTokenOut >= amountTokenOut) return;

        uint256 buidlToRedeem = amountTokenOut - contractBalanceTokenOut;

        buidl.safeIncreaseAllowance(address(buidlRedemption), buidlToRedeem);
        buidlRedemption.redeem(buidlToRedeem);
    }
```

## Impact

1. Lock of BUIDL token in RedemptionVaultWIthBUIDL contract.
2. Discrepancy between spec and code.

## Code Snippet

- https://github.com/sherlock-audit/2024-08-midas-minter-redeemer/blob/main/midas-contracts/contracts/RedemptionVaultWithBUIDL.sol#L164-L176

## Tool used

Manual Review

## Recommendation

Implement such feature.



## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/RedDuck-Software/midas-contracts/pull/68


**WangSecurity**

@pkqs90 could I get your clarification, this report is specifically about the minimum redemption limit and not about any other limitations, correct?

**pkqs90**

> @pkqs90 could I get your clarification, this report is specifically about the minimum redemption limit and not about any other limitations, correct?

Yes.

---
