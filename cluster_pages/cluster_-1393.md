# Cluster -1393

**Rank:** #451  
**Count:** 7  

## Label
Failure to persist reward token state leads to permanent loss of user rewards and irreversible contract failure due to missing state tracking and dynamic array misuse.

## Cluster Information
- **Total Findings:** 7

## Examples

### Example 1

**Auto Label:** Failure to persist reward token state leads to permanent loss of user rewards and irreversible contract failure due to missing state tracking and dynamic array misuse.  

**Original Text Preview:**

## Summary
Users might lose their `plenty`

## Vulnerability Details
When users mow, if a sop season has occurred since `lastUpdate`, they're calculated their `plenty` for each of the currently whitelisted tokens.

```solidity
        // If a Sop has occured since last update, calculate rewards and set last Sop.
        if (s.sys.season.lastSopSeason > lastUpdate) {
            address[] memory tokens = LibWhitelistedTokens.getWhitelistedWellLpTokens();
            for (uint i; i < tokens.length; i++) {
                s.accts[account].sop.perWellPlenty[tokens[i]].plenty = balanceOfPlenty(
                    account,
                    tokens[i]
                );
            }
            s.accts[account].lastSop = s.sys.season.lastSop;
        }
```

This could be problematic as if the user has had previous uncalculated plenty for a token that's now unwhitelisted, they won't be able to claim for it.

## Impact
Loss of funds

## Tools Used
Manual review

## Recommendations
Fix is non-trivial

---
### Example 2

**Auto Label:** Failure to persist reward token state leads to permanent loss of user rewards and irreversible contract failure due to missing state tracking and dynamic array misuse.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/108 

## Found by 
0x52

Some Aura pools have two sources of AURA. First from the booster but also as a secondary reward. This secondary reward is stash AURA that doesn't behave like regular AURA. Although properly accounted for in AuraSpell, it is not properly accounted for in WAuraPools, resulting in all deposits being unrecoverable. 

## Vulnerability Detail

[WAuraPools.sol#L413-L418](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L413-L418)

        uint256 rewardTokensLength = rewardTokens.length;
        for (uint256 i; i != rewardTokensLength; ) {
            IERC20Upgradeable(rewardTokens[i]).safeTransfer(
                msg.sender,
                rewards[i]
            );

When burning the wrapped LP token, it attempts to transfer each token to msg.sender. The problem is that stash AURA cannot be transferred like an regular ERC20 token and any transfers will revert. Since this will be called on every attempted withdraw, all deposits will be permanently unrecoverable.

## Impact

All deposits will be permanently unrecoverable

## Code Snippet

[WAuraPools.sol#L360-L424](https://github.com/sherlock-audit/2023-07-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L360-L424)

## Tool used

Manual Review

## Recommendation

Check if reward is stash AURA and send regular AURA instead similar to what is done in AuraSpell.



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**Kral01** commented:
> Needs PoC



**IAm0x52**

Escalate

This was incorrectly excluded. 

https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/spell/AuraSpell.sol#L243-L257

        for (uint256 i; i != rewardTokensLength; ) {
            address sellToken = rewardTokens[i];
            if (sellToken == STASH_AURA) sellToken = AURA;

            _doCutRewardsFee(sellToken);
            if (
                expectedRewards[i] != 0 &&
                !PSwapLib.swap(
                    augustusSwapper,
                    tokenTransferProxy,
                    sellToken,
                    expectedRewards[i],
                    swapDatas[i]
                )
            ) revert Errors.SWAP_FAILED(sellToken);

            /// Refund rest (dust) amount to owner
            _doRefund(sellToken);

            unchecked {
                ++i;
            }

AuraSpell requires the above code to prevent this. WAuraPools uses the exact same reward list and needs the same protections. The result is that funds will be permanently trapped, because the transfer will always fail.

**sherlock-admin2**

 > Escalate
> 
> This was incorrectly excluded. 
> 
> https://github.com/sherlock-audit/2023-07-blueberry/blob/7c7e1c4a8f3012d1afd2e598b656010bb9127836/blueberry-core/contracts/spell/AuraSpell.sol#L243-L257
> 
>         for (uint256 i; i != rewardTokensLength; ) {
>             address sellToken = rewardTokens[i];
>             if (sellToken == STASH_AURA) sellToken = AURA;
> 
>             _doCutRewardsFee(sellToken);
>             if (
>                 expectedRewards[i] != 0 &&
>                 !PSwapLib.swap(
>                     augustusSwapper,
>                     tokenTransferProxy,
>                     sellToken,
>                     expectedRewards[i],
>                     swapDatas[i]
>                 )
>             ) revert Errors.SWAP_FAILED(sellToken);
> 
>             /// Refund rest (dust) amount to owner
>             _doRefund(sellToken);
> 
>             unchecked {
>                 ++i;
>             }
> 
> AuraSpell requires the above code to prevent this. WAuraPools uses the exact same reward list and needs the same protections. The result is that funds will be permanently trapped, because the transfer will always fail.

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**Shogoki**

Seems valid
STASH AURA seems to be a valid `extraRewardToken` but transfer can only be called from the pool, which will actually transfer AURA instead. 
So if the WAURAPool tries to call transfer it will revert.

**hrishibhat**

Result:
Medium
Unique 
Considering this issue a valid medium

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [IAm0x52](https://github.com/sherlock-audit/2023-07-blueberry-judging/issues/108/#issuecomment-1694744574): accepted

---
### Example 3

**Auto Label:** Failure to persist reward token state leads to permanent loss of user rewards and irreversible contract failure due to missing state tracking and dynamic array misuse.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/128 

## Found by 
0x52
## Summary

pendingRewards pulls a fresh count of reward tokens each time it is called. This is problematic if reward tokens are ever removed from the the underlying Aura/Convex pools because it means that they will no longer be distributed and will be locked in the contract forever.

## Vulnerability Detail

[WAuraPools.sol#L166-L189](https://github.com/sherlock-audit/2023-04-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L166-L189)

        uint extraRewardsCount = IAuraRewarder(crvRewarder)
            .extraRewardsLength();
        tokens = new address[](extraRewardsCount + 1);
        rewards = new uint256[](extraRewardsCount + 1);

        tokens[0] = IAuraRewarder(crvRewarder).rewardToken();
        rewards[0] = _getPendingReward(
            stCrvPerShare,
            crvRewarder,
            amount,
            lpDecimals
        );

        for (uint i = 0; i < extraRewardsCount; i++) {
            address rewarder = IAuraRewarder(crvRewarder).extraRewards(i);
            uint256 stRewardPerShare = accExtPerShare[tokenId][i];
            tokens[i + 1] = IAuraRewarder(rewarder).rewardToken();
            rewards[i + 1] = _getPendingReward(
                stRewardPerShare,
                rewarder,
                amount,
                lpDecimals
            );
        }

In the lines above we can see that only tokens that are currently available on the pool. This means that if tokens are removed then they are no longer claimable and will be lost to those entitled to shares.

## Impact

Users will lose reward tokens if they are removed

## Code Snippet

[WAuraPools.sol#L152-L190](https://github.com/sherlock-audit/2023-04-blueberry/blob/main/blueberry-core/contracts/wrapper/WAuraPools.sol#L152-L190)

## Tool used

Manual Review

## Recommendation

Reward tokens should be stored with the tokenID so that it can still be paid out even if it the extra rewardToken is removed.

---
