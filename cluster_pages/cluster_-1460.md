# Cluster -1460

**Rank:** #239  
**Count:** 41  

## Label
Insufficient input validation and state transition flaws enable attackers to exploit financial invariants, cause unintended portfolio exposure, or manipulate auction outcomes through bid re-execution, zero-value bids, or unbounded challenge cycles.

## Cluster Information
- **Total Findings:** 41

## Examples

### Example 1

**Auto Label:** Insufficient input validation and state transition flaws enable attackers to exploit financial invariants, cause unintended portfolio exposure, or manipulate auction outcomes through bid re-execution, zero-value bids, or unbounded challenge cycles.  

**Original Text Preview:**

## Diﬃculty: High

## Type: Data Validation

## Description
During the am-AMM auction bidding process, a user vying to be a future manager can bid extremely low rent, even zero, in the absence of a current am-AMM manager or next bids, distorting the bidding process.

As depicted in figure 4.1, in the function `bid` in the AmAmm contract, when a user places a new bid, the rent is compared with the existing next bid rent and is set as the next bid if the rent is 10% more than the existing next bid rent. In the absence of an existing next bid, the user’s rent can be zero and still pass the sanity checks. As a result, future bidders may also opt for bidding low rent, aiming to be just above 10% of the previous rent.

```solidity
// update state machine
_updateAmAmmWrite(id);
// ensure bid is valid
// - manager can't be zero address
// - bid needs to be greater than the next bid by >10%
// - deposit needs to cover the rent for K hours
// - deposit needs to be a multiple of rent
// - payload needs to be valid
if (
    manager == address(0) || rent <=
    _nextBids[id].rent.mulWad(MIN_BID_MULTIPLIER(id)) || deposit < rent * K(id)
    || deposit % rent != 0 || !_payloadIsValid(id, payload)
) {
    revert AmAmm__InvalidBid();
}
```
*Figure 4.1: A snippet of the bid function in the AmAmm contract (src/AmAmm.sol#L75–L86)*

Moreover, as illustrated in figure 4.2, in the function `_stateTransitionWrite` in the contract AmAmm, a remarkably low next bid rent could still become the manager if there is no existing top bid, given that the rent must be greater than 10% of the existing top bid rent. This scenario allows the winning bidder to attain the manager’s role at a substantially low rent and earn significantly more in swap fees.

```solidity
// State D
// we charge rent from the top bid only until K epochs after the next bid was
// submitted
// assuming the next bid's rent is greater than the top bid's rent + 10%, otherwise
// we don't care about the next bid
bool nextBidIsBetter = nextBid.rent > topBid.rent.mulWad(MIN_BID_MULTIPLIER(id));
uint40 epochsPassed;
```
*Figure 4.2: A snippet of the _stateTransitionWrite function in the AmAmm contract (src/AmAmm.sol#L676–L680)*

## Recommendations
Short term, implement a minimum rent requirement to ensure that there is a lowest acceptable bid that a bidder must provide to qualify for consideration as the next bid.

---
### Example 2

**Auto Label:** Insufficient input validation and state transition flaws enable attackers to exploit financial invariants, cause unintended portfolio exposure, or manipulate auction outcomes through bid re-execution, zero-value bids, or unbounded challenge cycles.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-12-plaza-finance-judging/issues/891 

## Found by 
0xadrii, BADROBINX, Boy2000, Strapontin, bladeee, copperscrewer, i3arba, komane007, phoenixv110, shiazinho, t0x1c, y4y

### Summary

One of the condition for an auction to succeed is to have the [total bet of `reserveToken` be less than or equal to 90%](https://github.com/sherlock-audit/2024-12-plaza-finance/blob/main/plaza-evm/src/Auction.sol#L341) (or higher, set by admin) of the pool's balance to ensure tokens are transfered.

This calculation does not take into account the amount of fees claimable by the beneficiary, and can result in two issues :

- When an auction succeeds, it gets from the pool the amount of tokens users bet. This will drastically reduce the fees claimable from the beneficiary as it lowers the pool's balance, which is linked to fees calculation.

- If an auction should succeed by having the total of reserveToken bid being on the lower edge of the 90% of pool's token amount, and the fees are claimed, then the auction may fail if the reserveToken bid become higher than the newly calculated 90% to tokens in the pool.

### Root Cause

Auction does not includes the claimable fees when calculating the reserve amount it can receive

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

### Attack path 1

1. An auction is created and the condition for it to succeed are met (with an average total amount of `reserveToken` bid)
2. The amount of fees claimable are equal to X
3. The function `Auction::endAuction` is called and the auction succeeds, taking `reserveToken` from the pool
4. The amount of fees claimable are now lower than X

### Attack path 2

1. An auction is created and the condition for it to succeed are met (with a high total amount of `reserveToken` bid, near 90%)
2. The fees are claimed
3. The function `Auction::endAuction` is called and the auction ends in the state `FAILED_POOL_SALE_LIMIT` because the bids are higher than allowed amount of `reserveToken`

### Impact

Drastic reduction of fees claimable and potential auction ending in an unsuccessful state

### PoC

> To get the amount of fees claimable from the pool, set the visibility of the function `Pool::getFeeAmount` to public

Copy this poc in Auction.t.sol and run it

```solidity
    // forge test --mt test_auction_fees_1 -vvv
    function test_auction_fees_1() public {
        // We need to set the poolSaleLimit to 90% because it is set to 110% in the setUp
        uint256 poolSaleLimitSlot = 6;
        vm.store(address(auction), bytes32(poolSaleLimitSlot), bytes32(uint256(90)));
        console.log(auction.poolSaleLimit());

        // Set the fees at 10%
        vm.prank(governance);
        Pool(pool).setFee(100000);

        uint256 maxUSDCToBid = auction.totalBuyCouponAmount();
        // If we go beyond this value, endAuction will end in a failed state (FAILED_POOL_SALE_LIMIT)
        uint256 maxReserveTokenClaimable =
            (IERC20(auction.sellReserveToken()).balanceOf(pool) * auction.poolSaleLimit()) / 100;

        // 1. The auction can succeed, and will rewards users for half of the pool's claimable amount
        vm.startPrank(bidder);

        usdc.mint(bidder, maxUSDCToBid);
        usdc.approve(address(auction), maxUSDCToBid);

        auction.bid(maxReserveTokenClaimable / 2, maxUSDCToBid);

        vm.stopPrank();

        vm.warp(auction.endTime());

        // 2. Amount of fees claimable are equal to X
        // Set `getFeeAmount` to public to see its result value
        uint256 claimableFeesBefore = Pool(pool).getFeeAmount();
        console.log("claimableFeesBefore", claimableFeesBefore);

        // 3. `endAuction` put the auction in the succeed state
        auction.endAuction();
        assert(Auction.State.SUCCEEDED == auction.state());

        // 4. The amount of fees claimable are now lower than X
        uint256 claimableFeesAfter = Pool(pool).getFeeAmount();
        console.log("claimableFeesAfter ", claimableFeesAfter);

        assert(claimableFeesBefore > claimableFeesAfter);
    }

    // forge test --mt test_auction_fees_2 -vvv
    function test_auction_fees_2() public {
        // We need to set the poolSaleLimit to 90% because it is set to 110% in the setUp
        uint256 poolSaleLimitSlot = 6;
        vm.store(address(auction), bytes32(poolSaleLimitSlot), bytes32(uint256(90)));
        console.log(auction.poolSaleLimit());

        // Set the fees at 10%
        vm.startPrank(governance);
        Pool(pool).setFee(100000);
        Pool(pool).setFeeBeneficiary(governance);
        vm.stopPrank();

        uint256 maxUSDCToBid = auction.totalBuyCouponAmount();
        // If we go beyond this value, endAuction will end in a failed state (FAILED_POOL_SALE_LIMIT)
        uint256 maxReserveTokenClaimable = (IERC20(auction.sellReserveToken()).balanceOf(pool) * auction.poolSaleLimit()) / 100;

        // 1. The auction can succeed, and will rewards users for almost the pool's claimable amount
        vm.startPrank(bidder);

        usdc.mint(bidder, maxUSDCToBid);
        usdc.approve(address(auction), maxUSDCToBid);

        auction.bid(maxReserveTokenClaimable - 10, maxUSDCToBid);

        vm.stopPrank();

        vm.warp(auction.endTime());

        // 2. The fees are claimed
        vm.prank(governance);
        Pool(pool).claimFees();

        // 3. Ending the auction fails it
        auction.endAuction();
        assert(Auction.State.FAILED_POOL_SALE_LIMIT == auction.state());

        // Note that without the governance claiming fees, the auction would succeed
    }
```

Running them produces the following output :

```console
$ forge test --mt test_auction_fees_1 -vvv
[⠰] Compiling...
[⠒] Compiling 14 files with Solc 0.8.27
[⠰] Solc 0.8.27 finished in 24.60s
Compiler run successful!

Ran 1 test for test/Auction.t.sol:AuctionTest
[PASS] test_auction_fees_1() (gas: 448008)
Logs:
  90
  claimableFeesBefore 1369863013698630136986301369
  claimableFeesAfter  753424657534246575342465753

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 19.26ms (2.42ms CPU time)

Ran 1 test suite in 42.30ms (19.26ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)

...

$ forge test --mt test_auction_fees_2 -vvv
[⠰] Compiling...
No files changed, compilation skipped

Ran 1 test for test/Auction.t.sol:AuctionTest
[PASS] test_auction_fees_2() (gas: 467880)
Logs:
  90

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 19.09ms (2.16ms CPU time)

Ran 1 test suite in 39.14ms (19.09ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Mitigation

Include the claimable fees when calculating the total sell token limit at the end of an auction, or allocate an amount of tokens for an auction when the auction is created.

---
### Example 3

**Auto Label:** Insufficient input validation and state transition flaws enable attackers to exploit financial invariants, cause unintended portfolio exposure, or manipulate auction outcomes through bid re-execution, zero-value bids, or unbounded challenge cycles.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-12-plaza-finance-judging/issues/723 

The protocol has acknowledged this issue.

## Found by 
0x23r0, 0xDazai, 0xRaz, 0xc0ffEE, 0xmystery, AuditorPraise, Aymen0909, Benterkii, Boy2000, Chain-sentry, ChainProof, DenTonylifer, Hurley, KiroBrejka, Nave765, Ryonen, Saurabh\_Singh, Waydou, ZoA, aswinraj94, copperscrewer, elolpuer, evmboi32, farismaulana, gegul, krishnambstu, moray5554, novaman33, pashap9990, queen, rudhra1749, sl1, solidityenj0yer, t0x1c, zxriptor

### Summary

User can always inflate the [`totalSellReserveAmount`](https://github.com/sherlock-audit/2024-12-plaza-finance/blob/main/plaza-evm/src/Auction.sol#L151) variable to block the auction from being ended. This is an extremely and cheap attack to perform because the user practically loses nothing. He can call the [`Auction::bid`](https://github.com/sherlock-audit/2024-12-plaza-finance/blob/main/plaza-evm/src/Auction.sol#L125) function right before the end of the auction with some enormous `buyReserveAmount` as input. This will brick the money flow because he can do it over and over again for every auction, resulting in unprofitable investments for the people who hold `bondETH`


### Root Cause

`totalSellReserveAmount` being easily inflatable without any checks to prevent it 

### Internal Pre-conditions

User making a valid bid with enormous `buyReserveAmount` as input, right before the end of the auction

### External Pre-conditions

None

### Attack Path

1. User waits until for example 1 second before the end of the auction
2. Then he calls the bid function, making a valid bid with big `buyReserveAmount` input

With this the attack is already performed. After this happens and someone call the `Auction::endAuction` function, the auction will be in `FAILED_POOL_SALE_LIMIT` state because of this check:
```solidity
        } else if (
            totalSellReserveAmount >=
            (IERC20(sellReserveToken).balanceOf(pool) * poolSaleLimit) / 100
        ) 
```

### Impact

The money flow can be bricked and a user can purposefully bring every auction to `FAILED_POOL_SALE_LIMIT` for no cost at all, since he can just call the `Auction::claimRefund` function afterwords 

### PoC

None

### Mitigation

_No response_

---
