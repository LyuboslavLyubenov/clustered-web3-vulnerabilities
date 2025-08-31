# Cluster -1206

**Rank:** #458  
**Count:** 6  

## Label
Failure to validate critical preconditions before executing transfers or balance checks, leading to incorrect state transitions, fund locks, or misleading user feedback.

## Cluster Information
- **Total Findings:** 6

## Examples

### Example 1

**Auto Label:** Flawed permission checks enable unauthorized fund transfers or misuse of escrow states due to incorrect logic or missing validations.  

**Original Text Preview:**

##### Description

The `_onDeposit` internal function on the `RiverV1` contract uses `TRANSFER_MASK` and `DEPOSIT_MASK` variables to validate if user have permission to deposit ETH to the contract or transfer ETH to allowed addresses.

The `DEPOSIT_MASK` and `TRANSFER_MASK` variables are equal to `0x1` and `0x0` in order. The first `onlyAllowed` check tries to validate if the `_depositor` have permission for both deposit and transfer operations.

#### River.1.sol

```
(AllowlistAddress.get()).onlyAllowed(_depositor, DEPOSIT_MASK + TRANSFER_MASK);

```

However, addition of both these masks always returns `0x1` which is equals to `DEPOSIT_MASK`. As a result, use of this mathematical operation is unnecessary.

Code Location
-------------

#### River.1.sol

```
function _onDeposit(
    address _depositor,
    address _recipient,
    uint256 _amount
) internal override {
    SharesManagerV1._mintShares(_depositor, _amount);
    if (_depositor == _recipient) {
        (AllowlistAddress.get()).onlyAllowed(_depositor, DEPOSIT_MASK); // this call reverts if unauthorized or denied
    } else {
        (AllowlistAddress.get()).onlyAllowed(_depositor, DEPOSIT_MASK + TRANSFER_MASK); // this call reverts if unauthorized or denied
        (AllowlistAddress.get()).onlyAllowed(_recipient, TRANSFER_MASK);
        _transfer(_depositor, _recipient, _amount);
    }
}

```

##### Score

Impact: 1  
Likelihood: 1

##### Recommendation

**ACKNOWLEDGED:** This finding was acknowledged.

This is intended as it shows both rights are required to perform the action, but the fact that the mask is 0 means that it currently requires no special rights. This is a value that might be changed in the future depending on the protocol’s evolution and future needs.

---
### Example 2

**Auto Label:** Failure to validate critical preconditions before executing transfers or balance checks, leading to incorrect state transitions, fund locks, or misleading user feedback.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/21 

## Found by 
Aymen0909, KiroBrejka, ether\_sky, novaman33, sl1
## Summary
Seller's funds may remain locked in the protocol, because of revert on 0 transfer tokens.
In the README.md file is stated that the protocol uses every token with ERC20 Metadata and decimals between 6-18, which includes some revert on 0 transfer tokens, so this should be considered as valid issue!

## Vulnerability Detail
in the `AuctionHouse::claimProceeds()` function there is the following block of code:
```javascript
       uint96 prefundingRefund = routing.funding + payoutSent_ - sold_;
        unchecked {
            routing.funding -= prefundingRefund;
        }
        Transfer.transfer(
            routing.baseToken,
            _getAddressGivenCallbackBaseTokenFlag(routing.callbacks, routing.seller),
            prefundingRefund,
            false
        );
```
Since the batch auctions must be prefunded so `routing.funding` shouldn’t be zero unless all the tokens were sent in settle, in which case `payoutSent` will equal `sold_`. From this we make the conclusion that it is possible for `prefundingRefund` to be equal to 0. This means if the `routing.baseToken` is a revert on 0 transfer token the seller will never be able to get the `quoteToken` he should get from the auction.

## Impact
The seller's funds remain locked in the system and he will never be able to get them back.

## Code Snippet
The problematic block of code in the `AuctionHouse::claimProceeds()` function:
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L604-L613

`Transfer::transfer()` function, since it transfers the `baseToken`:
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/lib/Transfer.sol#L49-L68

## Tool used

Manual Review

## Recommendation
Check if the `prefundingRefund > 0` like this:
```diff
   function claimProceeds(
        uint96 lotId_,
        bytes calldata callbackData_
    ) external override nonReentrant {
        // Validation
        _isLotValid(lotId_);

        // Call auction module to validate and update data
        (uint96 purchased_, uint96 sold_, uint96 payoutSent_) =
            _getModuleForId(lotId_).claimProceeds(lotId_);

        // Load data for the lot
        Routing storage routing = lotRouting[lotId_];

        // Calculate the referrer and protocol fees for the amount in
        // Fees are not allocated until the user claims their payout so that we don't have to iterate through them here
        // If a referrer is not set, that portion of the fee defaults to the protocol
        uint96 totalInLessFees;
        {
            (, uint96 toProtocol) = calculateQuoteFees(
                lotFees[lotId_].protocolFee, lotFees[lotId_].referrerFee, false, purchased_
            );
            unchecked {
                totalInLessFees = purchased_ - toProtocol;
            }
        }

        // Send payment in bulk to the address dictated by the callbacks address
        // If the callbacks contract is configured to receive quote tokens, send the quote tokens to the callbacks contract and call the onClaimProceeds callback
        // If not, send the quote tokens to the seller and call the onClaimProceeds callback
        _sendPayment(routing.seller, totalInLessFees, routing.quoteToken, routing.callbacks);

        // Refund any unused capacity and curator fees to the address dictated by the callbacks address
        // By this stage, a partial payout (if applicable) and curator fees have been paid, leaving only the payout amount (`totalOut`) remaining.
        uint96 prefundingRefund = routing.funding + payoutSent_ - sold_;
++ if(prefundingRefund > 0) { 
        unchecked {
            routing.funding -= prefundingRefund;
        }
            Transfer.transfer(
            routing.baseToken,
            _getAddressGivenCallbackBaseTokenFlag(routing.callbacks, routing.seller),
            prefundingRefund,
            false
        );
++        }
       

        // Call the onClaimProceeds callback
        Callbacks.onClaimProceeds(
            routing.callbacks, lotId_, totalInLessFees, prefundingRefund, callbackData_
        );
    }
```



## Discussion

**nevillehuang**

#21, #31 and #112 highlights the same issue of `prefundingRefund = 0`

#78 and #97  highlights the same less likely issue of `totalInLessFees = 0`

All points to same underlying root cause of such tokens not allowing transfer of zero, so duplicating them. Although this involves a specific type of ERC20, the impact could be significant given seller's fund would be locked permanently

**sherlock-admin4**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/Axis-Fi/moonraker/pull/142


**10xhash**

> The protocol team fixed this issue in the following PRs/commits: [Axis-Fi/moonraker#142](https://github.com/Axis-Fi/moonraker/pull/142)

Fixed
Now Transfer library only transfers token if amount > 0

**sherlock-admin4**

The Lead Senior Watson signed off on the fix.

---
### Example 3

**Auto Label:** Failure to validate critical preconditions before executing transfers or balance checks, leading to incorrect state transitions, fund locks, or misleading user feedback.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-03-zap-protocol-judging/issues/58 

The protocol has acknowledged this issue.

## Found by 
bughuntoor, s1ce
## Summary
Users may not be able to claim tax refund

## Vulnerability Detail
Within TokenSale, upon depositing users, users have to pay tax. Then, users can receive a tax-free allocation - meaning they'll be refunded the tax they've paid on part of their deposit.

The problem is that due to a unnecessary require check, users cannot claim their tax refund, unless the token has oversold. 
```solidity
 function claim() external {
        checkingEpoch();
        require(
            uint8(epoch) > 1 && !admin.blockClaim(address(this)),
            "TokenSale: Not time or not allowed"
        );

        Staked storage s = stakes[msg.sender];
        require(s.amount != 0, "TokenSale: No Deposit"); 
        require(!s.claimed, "TokenSale: Already Claimed");

        uint256 left;
        (s.share, left) = _claim(s);
        require(left > 0, "TokenSale: Nothing to claim");  // @audit - problematic line 
        uint256 refundTaxAmount;
        if (s.taxAmount > 0) {
            uint256 tax = userTaxRate(s.amount, msg.sender);
            uint256 taxFreeAllc = _maxTaxfreeAllocation(msg.sender) * PCT_BASE;
            if (taxFreeAllc >= s.share) {
                refundTaxAmount = s.taxAmount;
            } else {
                refundTaxAmount = (left * tax) / POINT_BASE; // tax refund is on the wrong amount 
            }
            usdc.safeTransferFrom(marketingWallet, msg.sender, refundTaxAmount);
        }
        s.claimed = true;
        usdc.safeTransfer(msg.sender, left);
        emit Claim(msg.sender, left);
    }
```
```solidity

    function _claim(Staked memory _s) internal view returns (uint120, uint256) {
        uint256 left;
        if (state.totalPrivateSold > (state.totalSupplyInValue)) {
            uint256 rate = (state.totalSupplyInValue * PCT_BASE) /
                state.totalPrivateSold;
            _s.share = uint120((uint256(_s.amount) * rate) / PCT_BASE);
            left = uint256(_s.amount) - uint256(_s.share);
        } else {
            _s.share = uint120(_s.amount);
        }

        return (_s.share, left);
    }
```
`left` only has value if the token has oversold. Meaning that even if the user has an infinite tax free allocation, if the token has not oversold, they won't be able to claim a tax refund. 


## Impact
loss of funds

## Code Snippet
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L377

## Tool used

Manual Review

## Recommendation
Remove the require check 



## Discussion

**ZdravkoHr**

Escalate

This should not be a valid issue. The idea of the `claim` function is to let investors claim the surplus amount that is left after the ICO has ended, i.e when tokens `oversell`.

```solidity
 If the the demand is higher than supply, the number of tokens investors will receive is adjusted, and then the native token used to invest are partially refunded.
```

The following claim stated in the report is also wrong: 
```Then, users can receive a tax-free allocation - meaning they'll be refunded the tax they've paid on part of their deposit.```

Tax free allocation does not mean users will pay all the taxes and will be refunded later for the tax free amount. They are just not charged for the given amount from the very beginning of the deposit process. So they should not receive any refund.

This is evident from the way the tax is calculated in[`TokenSale._processPrivate()`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/TokenSale.sol#L231-L240)
```solidity
        if (sum > taxFreeAllcOfUser) {
            uint256 userTxRate = userTaxRate(sum, _sender);
            if (s.amount < taxFreeAllcOfUser) {
                userTaxAmount =
                    ((sum - taxFreeAllcOfUser) * userTxRate) /
                    POINT_BASE;
            } else {
                userTaxAmount = (amount * userTxRate) / POINT_BASE;
            }
        }
```

The reason why there is a tax refund logic in the `claim` function is because users that claim back `amount` of tokens will not have these tokens as deposited in the end of the ICO, therefore they should be refunded the tax they have paid for them.



**sherlock-admin2**

> Escalate
> 
> This should not be a valid issue. The idea of the `claim` function is to let investors claim the surplus amount that is left after the ICO has ended, i.e when tokens `oversell`.
> 
> ```solidity
>  If the the demand is higher than supply, the number of tokens investors will receive is adjusted, and then the native token used to invest are partially refunded.
> ```
> 
> The following claim stated in the report is also wrong: 
> ```Then, users can receive a tax-free allocation - meaning they'll be refunded the tax they've paid on part of their deposit.```
> 
> Tax free allocation does not mean users will pay all the taxes and will be refunded later for the tax free amount. They are just not charged for the given amount from the very beginning of the deposit process. So they should not receive any refund.
> 
> This is evident from the way the tax is calculated in[`TokenSale._processPrivate()`](https://github.com/sherlock-audit/2024-03-zap-protocol/blob/c2ad35aa844899fa24f6ed0cbfcf6c7e611b061a/zap-contracts-labs/contracts/TokenSale.sol#L231-L240)
> ```solidity
>         if (sum > taxFreeAllcOfUser) {
>             uint256 userTxRate = userTaxRate(sum, _sender);
>             if (s.amount < taxFreeAllcOfUser) {
>                 userTaxAmount =
>                     ((sum - taxFreeAllcOfUser) * userTxRate) /
>                     POINT_BASE;
>             } else {
>                 userTaxAmount = (amount * userTxRate) / POINT_BASE;
>             }
>         }
> ```
> 
> The reason why there is a tax refund logic in the `claim` function is because users that claim back `amount` of tokens will not have these tokens as deposited in the end of the ICO, therefore they should be refunded the tax they have paid for them.
> 
> 

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**koreanspicygarlic1**

This escalation is also plain wrong, watson has not properly understood the design of the system.

**vsharma4394**

If the above reasoning is wrong then my issue #159  which has been invalidated becomes a valid issue,so can someone escalate that too. @Hash01011122 please look into it carefully please.

**vsharma4394**

Hardcoding the value of taxfreeAllocation to zero implies that the protocol doesn't allow for taxFreeAllocation to occur as of now. So this directly implies that while claiming also taxFreeAmount should not be taken into consideration while refunding the amount to the user.


**Hash01011122**

Well I don't thoroughly understand the basis of this escalation as it is clearly mentioned in the codebase how tax system is calculated, where the tax-free allocation isn't zero. 

**vsharma4394**

@Hash01011122 the following loc in _processPrivate function which is called when user deposits usdc has caused different understanding of he code 
```solidity
 uint256 taxFreeAllcOfUser = 0; // hardcode zero - all pools have ax

        uint256 userTaxAmount;

        if (sum > taxFreeAllcOfUser) {
            uint256 userTxRate = userTaxRate(sum, _sender);
            if (s.amount < taxFreeAllcOfUser) {
                userTaxAmount =
                    ((sum - taxFreeAllcOfUser) * userTxRate) /
                    POINT_BASE;
            } else {
                userTaxAmount = (amount * userTxRate) / POINT_BASE;
            }
        }
```
Due to hardcoded value of taxFreeAllocation as zero ,user tax amount is calculated as follows
userTaxAmount = (amount * userTxRate) / POINT_BASE i.e while depositing users have to pay tax irrespective of having taxFreeAllocation. If logic is never executed. So while claiming also taxFreeAllocation should also not be taken into account so as be consistent with the code. (Making taxFreeAllocation to zero is intended design.)

I think confusion has arised from considering taxFreeAllocation but it is considered as zero from starting due to hardcoded value of taxFreeAllocation as zero.

Asking from sponsers is the best way to deal with all the issue related to claim function and tax related issue.

**ZdravkoHr**

The issue is not that tax free allocation is 0. Even if it wasn't and the firsr if was entered, the user would not have been taxed for it. That's why I believe a refund should not be made

**niketansainiantier**

we are not giving a refund for the tax if a sale does not reach the hard cap.

**vsharma4394**

@niketansainiantier so this issue should be invalid right?

**Hash01011122**

Well I agree with what @vsharma4394 has mentioned above and came to the same conclusion when I revisited codebase. @ZdravkoHr and even sponsors have confirmed this one. This is valid finding.

**vsharma4394**

@Hash01011122 i dont think that this issue is valid because @niketansainiantier clearly mentioned the following 
```we are taking the tax on the whole invested amount including all allocations, including the whitelist. So will refund the only tax on left amount(extra Amount).```
Now if token oversells tax is refunded based on the extra amount which should not consider taxFreeAllocation as said by the sponsers. Thus in claim function 
```solidity
if (s.taxAmount > 0) {
            uint256 tax = userTaxRate(s.amount, msg.sender);
            uint256 taxFreeAllc = _maxTaxfreeAllocation(msg.sender) * PCT_BASE;
            if (taxFreeAllc >= s.share) {
                refundTaxAmount = s.taxAmount;
            } else {
                refundTaxAmount = (left * tax) / POINT_BASE; // tax refund is on the wrong amount 
            }
            usdc.safeTransferFrom(marketingWallet, msg.sender, refundTaxAmount);
        }
```
RefundTaxAmount should always be  refundTaxAmount = (left * tax) / POINT_BASE because taxFreeAlloc is hardcoded as zero initially which clearly indicates protocol doesn't allow for taxFreeAllocation as of now.

**Evert0x**

I believe the escalation should be rejected the issue should stay as is.

Users should be able to get a tax refund on their tax-free allocation. 

> **Tax free allocation does not mean users will pay all the taxes and will be refunded later for the tax free amount.** They are just not charged for the given amount from the very beginning of the deposit process. So they should not receive any refund.

This is not true, would like to see a link to a public message or to a code comment as a counter argument 

**vsharma4394**

> I believe the escalation should be rejected the issue should stay as is.
> 
> Users should be able to get a tax refund on their tax-free allocation.
> 
> > **Tax free allocation does not mean users will pay all the taxes and will be refunded later for the tax free amount.** They are just not charged for the given amount from the very beginning of the deposit process. So they should not receive any refund.
> 
> This is not true, would like to see a link to a public message or to a code comment as a counter argument

@Evert0x i agree the above reasoning is incorrect, please look at my reasoning and then decide

**Evert0x**

@vsharma4394  My understanding that the codebase is taxing all deposited and provide a tax-return on unallocated part + tax-free allocation. Which makes this issue a valid issue.

So if the token doesn't oversell, we should still take into account the tax free allocation. 

I don't understand how your previous comment provides an argument against that. 

**vsharma4394**

> @vsharma4394 My understanding that the codebase is taxing all deposited and provide a tax-return on unallocated part + tax-free allocation. Which makes this issue a valid issue.
> 
> So if the token doesn't oversell, we should still take into account the tax free allocation.
> 
> I don't understand how your previous comment provides an argument against that.

This would most probably because of the protocol design @niketansainiantier  can answer it the best. 

**Evert0x**

Result:
High
Has Duplicates

**sherlock-admin3**

Escalations have been resolved successfully!

Escalation status:
- [ZdravkoHr](https://github.com/sherlock-audit/2024-03-zap-protocol-judging/issues/58/#issuecomment-2025286194): rejected

---
