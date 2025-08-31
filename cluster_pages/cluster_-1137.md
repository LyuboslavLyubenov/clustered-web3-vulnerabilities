# Cluster -1137

**Rank:** #345  
**Count:** 18  

## Label
**Blacklist exploitation leading to reward denial, fund locking, and unfair distributions via transaction reverts or unvalidated transfers.**

## Cluster Information
- **Total Findings:** 18

## Examples

### Example 1

**Auto Label:** **Blacklisting of addresses triggers irreversible fund locks or liquidation failures via centralized token controls.**  

**Original Text Preview:**

```solidity
    function cancelMint(uint256 _id) external mintRequestExist(_id) {
// ..snip
        request.state = State.CANCELLED;
        IERC20 depositedToken = IERC20(request.token);
|>        depositedToken.safeTransfer(request.provider, request.amount);
        emit MintRequestCancelled(_id);
    }
        function cancelBurn(uint256 _id) external burnRequestExist(_id) {
// ..snip
        request.state = State.CANCELLED;
        IERC20 issueToken = IERC20(ISSUE_TOKEN_ADDRESS);
|>        issueToken.safeTransfer(request.provider, request.amount);

        emit BurnRequestCancelled(_id);
    }
```

As seen, the cancellation functions `ExternalRequestsManager::cancelMint()` and `cancelBurn()` directly transfer funds to `msg.sender` using `safeTransfer`:

Now, currently, usd-fun only support USDC/USDT (which are both blacklistable stablecoins), knowing that transfers to blacklisted addresses will revert. This creates an irrecoverable state where:

1. Blacklisted provider, who had initially placed a mint/burn request, attempts to call `cancelMint()/cancelBurn()`
2. `safeTransfer` fails due to blacklist check in token contract
3. Transaction reverts, keeping request in `CREATED` state
4. Protocol cannot process cancellation the cancellation for users

Allow the providers to pass in a recipient, this can be done by attaching a new parameter to the cancellation functions:

```solidity
function cancelMint(uint256 _id, address _recipient) external {
    // ...
    depositedToken.safeTransfer(_recipient, request.amount);
}

function cancelBurn(uint256 _id, address _recipient) external {
    // ...
    issueToken.safeTransfer(_recipient, request.amount);
}
```

---
### Example 2

**Auto Label:** **Blacklisting of addresses triggers irreversible fund locks or liquidation failures via centralized token controls.**  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-08-sentiment-v2-judging/issues/284 

## Found by 
000000, iamnmt
## Summary
Liquidations will revert if a position has been blacklisted for USDC
## Vulnerability Detail
Upon liquidations, we call the following function:
```solidity
function _transferAssetsToLiquidator(address position, AssetData[] calldata assetData) internal {
        // transfer position assets to the liquidator and accrue protocol liquidation fees
        uint256 assetDataLength = assetData.length;
        for (uint256 i; i < assetDataLength; ++i) {
            // ensure assetData[i] is in the position asset list
            if (Position(payable(position)).hasAsset(assetData[i].asset) == false) {
                revert PositionManager_SeizeInvalidAsset(position, assetData[i].asset);
            }
            // compute fee amt
            // [ROUND] liquidation fee is rounded down, in favor of the liquidator
            uint256 fee = liquidationFee.mulDiv(assetData[i].amt, 1e18);
            // transfer fee amt to protocol
            Position(payable(position)).transfer(owner(), assetData[i].asset, fee);
            // transfer difference to the liquidator
            Position(payable(position)).transfer(msg.sender, assetData[i].asset, assetData[i].amt - fee);
        }
    }
```
As seen, we call `transfer()` on the `Position` contract which just transfers the specified amount of funds to the provided receiver. As mentioned in the contest README, USDC will be whitelisted for the protocol. If the `position` address is blacklisted for USDC, this transcation would fail and the liquidation for that user would brick. The user in charge of that position could increase his chance of getting blacklisted by using the `exec()` function which calls a particular function on a target (both have to be whitelisted by an owner). If they do malicious stuff and even worse, manage to find a vulnerability that they can exploit on the allowed target, they might get blacklisted which would brick liquidations for them, especially if their only deposited collateral token is USDC.

Even worse, every user can call `addToken()` for USDC without having to deposit any USDC nor to have any USDC balance making this attack free, the only thing the user needs to make happen is to get blacklisted.
## Impact
Liquidations will revert if a position has been blacklisted for USDC. Likelihood - low, impact - high
## Code Snippet
https://github.com/sherlock-audit/2024-08-sentiment-v2/blob/25a0c8aeaddec273c5318540059165696591ecfb/protocol-v2/src/PositionManager.sol#L478
## Tool used

Manual Review

## Recommendation
Fix is not trivial but an option is implementing a try/catch block and additional checks in order to not make the liquidator unwillingly repay the debt while not receiving the collateral.



## Discussion

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**z3s** commented:
> #258



**samuraii77**

Escalate

The judge has said that the issue is not valid as the protocol is trusted and won't get blacklisted. That is not important for my issue at all, a user could get his position blacklisted using the way explained in my report.

**sherlock-admin3**

> Escalate
> 
> The judge has said that the issue is not valid as the protocol is trusted and won't get blacklisted. That is not important for my issue at all, a user could get his position blacklisted using the way explained in my report.

The escalation could not be created because you are not exceeding the escalation threshold.

You can view the required number of additional valid issues/judging contest payouts in your Profile page,
in the [Sherlock webapp](https://app.sherlock.xyz/audits/).


**AtanasDimulski**

Escalate,
Per the above [comment](https://github.com/sherlock-audit/2024-08-sentiment-v2-judging/issues/284#issuecomment-2351851715)

**sherlock-admin3**

> Escalate,
> Per the above [comment](https://github.com/sherlock-audit/2024-08-sentiment-v2-judging/issues/284#issuecomment-2351851715)

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**cvetanovv**

I agree that the point here is that tokens can be blacklisted not by the protocol but by the tokens themselves if they do malicious things. 

The owner of these tokens will still be able to do borrowing but will not be able to be liquidated.

Planning to accept the escalation, duplicate this issue with #258, and make a valid Medium.

**WangSecurity**

Result: 
Medium
Has duplicates

**sherlock-admin2**

Escalations have been resolved successfully!

Escalation status:
- [AtanasDimulski](https://github.com/sherlock-audit/2024-08-sentiment-v2-judging/issues/284/#issuecomment-2352308356): accepted

**0xjuaan**

Hi @WangSecurity @cvetanovv sorry this is a late message, but I don't think its valid to say that the position contract will be blacklisted. The allowed function calls that can be called via `exec()` are given [here](https://gist.github.com/ruvaag/58c9fc2e5c139451c83c21fda27b77a2). 

These are normal DeFi functions on GMX and Pendle, so USDC will not blacklist the position upon calling any of these since they can't do anything malicious.

[Here](https://docs.sherlock.xyz/audits/judging/judging#vii.-list-of-issue-categories-that-are-not-considered-valid) are the sherlock rules regarding blacklists:
> Contract / Admin Address Blacklisting / Freezing: If a protocol's smart contracts or admin addresses get added to a "blacklist" and the functionality of the protocol is affected by this blacklist, this is not considered a valid issue.

The only exception is when a pre-blacklisted address can be used to cause harm to the protocol, like [this example](https://github.com/sherlock-audit/2022-11-opyn-judging/issues/219). In this issue however, the blacklist occurs on a protocol contract (the Position contract), so it should be invalidated.



**cvetanovv**

@0xjuaan Yeah, you're right in this case, maybe there's no harm for the protocol.

@samuraii77 Can you give your opinion?

**samuraii77**

@cvetanovv, firstly, the rule cited is not of significance here as there is clearly harm for everyone if the position gets blacklisted, liquidations won't be possible for that position which is detrimental. 

Now, the question would be whether the position can get blacklisted. As explained in the report, there is an `exec()` function which allows freely calling a few functions on a few target addresses.
>These are normal DeFi functions on GMX and Pendle

Yes, they are normal functions, never claimed otherwise and not sure what else would they be other than normal functions.

>so USDC will not blacklist the position upon calling any of these since they can't do anything malicious.

Yes, USDC will not blacklist the position if the user __can't__ do anything malicious. However, why would you assume he can't do anything malicious? We have a total of 6 functions currently allowed (very likely that this would increase in the future). As with how common hacks are in DeFi, we can't assume that there is nothing that can be abused in these 6 functions. It is definitely not an unlikely assumption that using those 6 functions, a user would be able to do malicious stuff which would cause his position to get blacklisted. Furthermore, these functions allow completely handling control of the execution using native ETH so that essentially gives the user the ability to call any contract they would like.

If we take a look at the rules to identify a medium severity issue, we are clearly meeting those requirements with this issue:
>Causes a loss of funds but requires certain external conditions or specific states

This issue is definitely not of a high likelihood, that's why I have submitted it as a medium. It is of low likelihood (but definitely not low enough to be assumed to not happen) but high impact, a medium severity is perfectly appropriate.

I also believe that any disagreements about the validity of the issue must have been made while the issue was escalated, not afterwards.

**0xjuaan**

@samuraii77 do you have a concrete example of a malicious action that can occur through exec() that would cause the position to be blacklisted?

**0xjuaan**

@cvetanovv @WangSecurity  just pinging as a reminder so this does not go unnoticed since the escalation was already resolved.

**cvetanovv**

> @samuraii77 do you have a concrete example of a malicious action that can occur through exec() that would cause the position to be blacklisted?

@samuraii77 Can you respond to this comment by @0xjuaan?

**samuraii77**

I wouldn't share such a malicious action in a public repo for everyone to see, if I knew of a vulnerability in those target contracts, I would disclose it to the respective protocols, not here. 

Either way, I don't think that's of significance here, whether I know of such a malicious action or not does not matter, we are looking for issues in this particular protocol, not in external protocols. The issue is in this protocol, assuming the position is not blacklisted when tokens like USDC are to be used, to be precise. Not only such an assumption was made but users have the opportunity to interact with external protocols using the `exec()` functionality as already mentioned which significantly increases chances of the position getting blacklisted. Users can also freely add `USDC` as their position assets without having to deposit any tokens due to the protocol design, this also decreases the risk the users has to take in order to try and make this attack.

I believe we clearly fall under the following rule:
>Causes a loss of funds but requires certain external conditions or specific states


**neko-nyaa**

The "Position" contract belongs to the user, not to the protocol. The protocol gives the contract to the user so that they can create a position within the protocol. Since the admin does not have control over the Position contract, it can no longer be considered a "protocol's smart contracts".

The rule about blacklisting causing harm also never mentions the pre-blacklisting as a requirement. In this case, if the Position is blacklisted, then the undercollateralized borrower "used a blacklisted address" and "caused harm to a protocol functioning". The protocol functioning is liquidation, the harm is the function being denied, the impact is undercollateralized i.e. bad debt borrowing.

**cvetanovv**

I agree with @samuraii77, and I will keep this issue valid.

However, if a user's position address is blacklisted, it can indeed harm the protocol by preventing liquidations, which could result in bad debt. 

**0xjuaan**

@cvetanovv what about the sherlock rule? 

> Contract / Admin Address Blacklisting / Freezing: If a protocol's smart contracts or admin addresses get added to a "blacklist" and the functionality of the protocol is affected by this blacklist, this is not considered a valid issue.

The Position contract is deployed by the protocol, using the protocol's implementation contract, so it cannot be assumed that the contract will be blacklisted. That's why the rule was made. Otherwise I can say that if a Pool gets blacklisted, users USDC will be stuck forever, and that should be a valid issue.

cc @WangSecurity

**samuraii77**

Just the fact that it was deployed by the protocol does not make it the protocol's contract. The position is in full control of the position owner who is not a trusted entity, the protocol has absolutely nothing to do with it and has no control over it. 

As a matter of fact, it is not even deployed by the protocol. It is deployed by the PositionManager contract at the will of a regular user through __the user__ calling the respective function for creating a new position.

**cvetanovv**

> @cvetanovv what about the sherlock rule?
> 
> > Contract / Admin Address Blacklisting / Freezing: If a protocol's smart contracts or admin addresses get added to a "blacklist" and the functionality of the protocol is affected by this blacklist, this is not considered a valid issue.
> 
> The Position contract is deployed by the protocol, using the protocol's implementation contract, so it cannot be assumed that the contract will be blacklisted. That's why the rule was made. Otherwise I can say that if a Pool gets blacklisted, users USDC will be stuck forever, and that should be a valid issue.
> 
> cc @WangSecurity

You have not quoted the whole rule, here is the further part:
> However, there could be cases where an attacker would use a blacklisted address to cause harm to a protocol functioning. [Example(Valid)](https://github.com/sherlock-audit/2022-11-opyn-judging/issues/219)

Watson has given an example of how being blacklisted can hurt the protocol (getting into bad debt).

**0xjuaan**

@cvetanovv if you look at that example, it involves using a pre-blacklisted address which is a non-protocol address.

this issue does not fit that example since this involves protocol smart contracts being blacklisted.

**samuraii77**

As I mentioned, this is not at all a protocol smart contract. It is deployed at the will of a user and the user is the owner of the contract. I don't understand why you keep saying that this is the protocol's smart contract when they have no control over it. They are not even the ones deploying it, it is deployed by the `PositionManager` contract at the will of __the user__ and the ownership is solely the __user's__, there is absolutely no correlation between the protocol and the `Position` contract. 

**0xjuaan**

@samuraii77 it's a protocol smart contract because it's deployed by a protocol smart contract, with implementation defined by the protocol- the fact that a user triggers it does not matter. there's no way to get the contract blacklisted because it cant do anything malicious.

i'm not gonna speak on this issue anymore, if it gets validated its a failure to understand the guidelines in the rules.

**cvetanovv**

@0xjuaan @samuraii77 To be fair, I'll ask HoJ to look at the issue and give his opinion. I may be misinterpreting the rule.

**cvetanovv**

After reviewing the situation and considering HoJ's feedback, I agree that this issue should be marked as invalid.

The core argument that the Position contract could get blacklisted lacks a concrete, realistic example of how this could happen through the currently allowed functions. Without a valid path to demonstrate how the `exec()` function or external protocol interactions could lead to blacklisting, this scenario remains highly speculative.

Furthermore, USDC's [blacklist policy](https://www.circle.com/hubfs/Blog%20Posts/Circle%20Stablecoin%20Access%20Denial%20Policy_pdf.pdf) targets only severe cases, and there is no evidence to suggest that a Position contract, triggered by standard DeFi interactions, would fall under this category. 

With only a small number of addresses blacklisted by USDC, this situation seems too rare to consider a genuine threat. There are only 12 addresses added since the beginning of the year. With millions of users using USDC, this can be a rare edge case - https://dune.com/phabc/usdc-banned-addresses. For the above reasons, we consider the issue Low severity.

We will reject escalation, and this issue will remain invalid.

**samuraii77**

I believe the blacklist policy shown further makes my issue more valid. We can see that addresses can get blacklisted not only if they commit on-chain but also based on different requests from jurisdictions, countries, governments, etc. which only increases the likelihood and doesn't decrease it, thus the `exec()` functionality allowing malicious actions is not a prerequisite to the issue, it is only a boost to the likelihood.

For example, by searching on the internet, here are some sitautions where a blacklist can occur that is not related to an on-chain hack:
- law enforcement requests
- sanctions compliance (sanctioned individuals, sanctioned countries)
- wallets associated with financing illegal activities
- addresses linked to ransomware attacks
- money laundering operations
- regulatory compliance
- and more

Nothing stops such a person conducting illegal activities from being linked to the position contract. As the position contract is fully in control and possession of an untrusted user, we can assume that a link between the position contract and such an entity is probable.

An address can even get blacklisted by stealing 1,000,000$ from a smart contract and then send the funds over to the position. As USDC/Circle does not want those funds to be transferred, that will cause the position to get blacklisted. If that wasn't the case and USDC wouldn't blacklist such positions, then that means that every attacker can abuse this and transfer their funds to such a position to avoid getting blacklisted - this is clearly not the case and such positions will get the same treatment as a regular address. Thus, there are many different situations where a position can get blacklisted on top of the `exec()` functionality which makes it even more likely. It is definitely not an imaginary event but an actual scenario that can occur. All issues related to USDC have a low likelihood by default, it is not much different here.

Furthermore, the amount of number of addresses getting blacklisted by USDC is not a valid argument. I am not claiming that it happens often, otherwise that would be a high severity issue. The rules are clear regarding this, if a blacklist harms users other than the one blacklisted, it is valid - that is precisely the scenario here.

The protocol design is clearly flawed in terms of that scenario and it allows the discussed scenario to occur, this should be a valid medium severity issue.

**cvetanovv**

After careful review, I agree with the reasoning presented by @samuraii77.

The potential for a position to be blacklisted can arise from various realistic scenarios beyond just on-chain activities. Legal requests, sanctions compliance, and regulatory measures can all contribute to blacklisting risks. According to the rules, if a blacklisting harms the functioning of the protocol and not just the affected individual, it is considered a valid issue.

In this case, the user could intentionally get blacklisted, potentially causing harm to the protocol's liquidation process. This aligns with the rule that states, "there could be cases where an attacker would use a blacklisted address to cause harm to a protocol functioning."

He has control over getting himself blacklisted and causing harm to the protocol, which according to the rules is a valid issue.

My decision is to keep the issue as it is.

**0xjuaan**

>  According to the rules, if a blacklisting harms the functioning of the protocol and not just the affected individual, it is considered a valid issue.

@cvetanovv If you read the rule again, it says "If a protocol's smart contracts or admin addresses get added to a "blacklist" and the functionality of the protocol is affected by this blacklist, this is NOT considered a valid issue." which is the exact opposite of your statement.




**samuraii77**

@0xjuaan, you are misinterpreting the rule as I told you a few times. Judging by your logic, an account abstraction contract, that is fully in control of a user, is a protocol's smart contract as it gets deployed by a factory even though it is completely in control of a user.

The rule is about contracts that are in control of the protocol, for example a protocol contract owning USDC to lend out to users getting blacklisted, that would not be a valid submission as that contract is the protocol's.

---
### Example 3

**Auto Label:** **Blacklist exploitation leading to reward denial, fund locking, and unfair distributions via transaction reverts or unvalidated transfers.**  

**Original Text Preview:**

**Severity**: Low

**Status**: Resolved

**Description**

In Contract WaveContract.sol, the method executeRuffle(...) sends rewards to all the winners in one go. 

If any of the winner addresses are unable to receive the token rewards due to being blacklisted (e.g. USDT), then the transfer of rewards to all users will be reverted. This will also be possible if any user sends it’s NFT token to one or more blacklisted addresses.

This will lead to a DoS attack for all valid users as well.

**Recommendation**: 

Add a method to allow the winner to withdraw rewards.

---
