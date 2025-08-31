# Cluster -1409

**Rank:** #453  
**Count:** 7  

## Label
Race conditions in balance updates lead to misleading event emissions and inconsistent state visibility, enabling erroneous transfers or fund loss through improper ordering or delayed balance adjustments.

## Cluster Information
- **Total Findings:** 7

## Examples

### Example 1

**Auto Label:** Race conditions in balance updates lead to misleading event emissions and inconsistent state visibility, enabling erroneous transfers or fund loss through improper ordering or delayed balance adjustments.  

**Original Text Preview:**

## Description

The SendToken amount logged in its event can be incorrect in some circumstances. The SendToken action can accept `type(uint256).max` as a special amount input. When received, it will transfer the full balance of the Safe wallet for that token. This is not reflected in the event logging emitted alongside the transfer, as it uses `inputData.amount`, which would still be `type(uint256).max`.

## Recommendations

Alter the logic of `executeAction()` such that the adjustment for a `type(uint256).max` amount is done prior to calling `_sendToken()` so it is also reflected in the event logging.

## Resolution

The SendToken action’s event now reports the correct amount of tokens sent.

---
### Example 2

**Auto Label:** Race conditions in balance updates lead to misleading event emissions and inconsistent state visibility, enabling erroneous transfers or fund loss through improper ordering or delayed balance adjustments.  

**Original Text Preview:**

## Summary

Current version of Beanstalk deployed on Mainnet has following logic of order creation in Market:

Order on Market means that order creator wants to buy Pod for Beans. During creation Market transfers Beans from orderer to Beanstalk. Problem is that those transferred Beans are not reflected on User's internal balance.

It means after migration these Beans won't be returned because it is not internal balance.

## Vulnerability Details

Here you can see that Market transfers Beans from user via function `LibTransfer.receiveToken()`:
<https://github.com/BeanstalkFarms/Beanstalk/pull/909/files#diff-38bfddf2eaaa5a2f714bcff17d7bef97c83eed773fb2a0e51c6e327eee98b839L113-L133>

As you can see these Beans are not reflected in internal balance:

```solidity
    function receiveToken(
        IERC20 token,
        uint256 amount,
        address sender,
        From mode
    ) internal returns (uint256 receivedAmount) {
        if (amount == 0) return 0;
        if (mode != From.EXTERNAL) {
            receivedAmount = LibBalance.decreaseInternalBalance(
                sender,
                token,
                amount,
                mode != From.INTERNAL
            );
            if (amount == receivedAmount || mode == From.INTERNAL_TOLERANT) return receivedAmount;
        }
        uint256 beforeBalance = token.balanceOf(address(this));
        token.safeTransferFrom(sender, address(this), amount - receivedAmount);
        return receivedAmount.add(token.balanceOf(address(this)).sub(beforeBalance));
    }
```

So it means Users will lose those Beans during migration because it won't be refunded via contract ReseedInternalBalances.sol

## Impact

Users who have open orders on Market during migration will lose their Beans submitted during order creation.

## Tools Used

Manual Review

## Recommendations

Migrate orderers' Beans submitted to Beanstalk.

---
### Example 3

**Auto Label:** Inconsistent state updates due to improper balance synchronization and loop-order exploits lead to unauthorized token transfers and incorrect balance tracking.  

**Original Text Preview:**

**Severity:** Critical

**Path:** ERC20Pods.sol#L85-L96

**Description:** 

In the function `_removeAllPods` contract iterates over the delegated pods
and burns the delegation balances via calling `updateBalances` on pods,
the bug arises since the balance is saved once on line 87, and an illicit actor
can use both a valid pod and a rogue pod. Thus the first call to the rogue
pod can be used to make a highjack execution flow and change the balance
between the pod removals.

Attack scenario:
1. The illicit actor sends out almost all his tokens to a vault contract
(pre-deployed for the attack), such that he’s left with only 1 token
(1/10**decimals of the token) on its balance
2. Adds both valid and rogue pods via calling `addPod`
3. Calls `removeAllPods`
4. balance variable is initialised with value 1, and the rogue pod will be
called first to update the balances
5. Rogue pod hijacks the flow and transfers back all of the tokens back to
the illicit actor
6. The next iteration will call the valid pod with balance = 1 instead of the
new balance, so the actor’s pod balance will be changed only by 1,
and the pod will be removed
7. The illicit actor again calls `addPod` with the valid pod, which mints pod
tokens.

```
function _removeAllPods(address account) internal virtual {
       address[] memory items = _pods[account].items.get();
       uint256 balance = balanceOf(account);
       unchecked {
           for (uint256 i = items.length; i > 0; i--) {
               if (balance > 0) {
                   _updateBalances(items[i - 1], account, address(0), balance);
               }
               _pods[account].remove(items[i - 1]);
           }
       }
   }
```

**Remediation:**  Update the balance in the loop or use balanceOf in
_updateBalances function.

**Status:**  Fixed

- - -

---
