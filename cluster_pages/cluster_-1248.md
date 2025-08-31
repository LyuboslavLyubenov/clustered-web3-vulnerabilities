# Cluster -1248

**Rank:** #58  
**Count:** 300  

## Label
Improper state indexing and value handling in value-based operations enable incorrect value transfers, balance inflation, and fee bypass through misaligned or unmasked state access and flawed input-output accounting.

## Cluster Information
- **Total Findings:** 300

## Examples

### Example 1

**Auto Label:** Insufficient state validation and unauthorized state manipulation enable attackers to bypass critical checks, leading to unauthorized transfers, auction manipulation, or protocol devaluation.  

**Original Text Preview:**

The `Folio.bid()` function enforces that bids specify a strict `sellAmount` such that `minSellAmount == sellAmount == maxSellAmount`.

```solidity
function bid(
    //...
    uint256 sellAmount,
    //...
) external nonReentrant notDeprecated sync returns (uint256 boughtAmt) {
    Auction storage auction = auctions[auctionId];

    // checks auction is ongoing and that boughtAmt is below maxBuyAmount
    (, boughtAmt, ) = _getBid(
        --- SNIPPED ---
        sellAmount,             //> minSellAmount
        sellAmount,             //> maxSellAmount
        maxBuyAmount
    );

    --- SNIPPED ---
}
```

The available sell balance is checked at execution time in `RebalanceLib::getBid()`:

```solidity
require(sellAvailable >= params.minSellAmount, IFolio.Folio__InsufficientSellAvailable());
```

This creates a scenario where any change to the protocol balance state between bid creation and execution can cause bids to revert, which can occur through:

1. Other bids: Each successful bid could reduce the `sellToken` balance.
2. Redemptions: These affect both the `sellToken` balance and the `sellLimitBal` calculation, as calculated below:
   ```solidity
    uint256 sellLimitBal = Math.mulDiv(sellLimit, params.totalSupply, D27, Math.Rounding.Ceil);
    uint256 sellAvailable = params.sellBal > sellLimitBal ? params.sellBal - sellLimitBal : 0;
   ```

This issue can be exploited through front-running or can occur naturally through normal protocol usage, blocking bids from executing.
As auction prices decay over time, the protocol could miss favorable execution opportunities.

**Consider the following scenario**:

#### Initial state

0. `totalSupply()` 1000 shares, rebalance targeting reduction from 1000 ETH to 500 ETH, and buy 1,500,000 USDT:

   - 1 BU now consist of 1 ETH 0 USDT.
   - 1 BU traget to consist of 0.5 ETH, 1500 USDT.

1. User A submits a transaction for a full bid: 500 ETH → 1,500,000 USDT at start price (most favorable price).
2. User B front-runs with a minimal bid: 0.0003333 ETH → 1 USDT.
3. The sell token balance decreases to 999.9996667 ETH.
4. The `sellAvailable` becomes 499.9996667 ETH (999.9996667 - 500 ETH).
5. When User A's transaction executes, the check `499.9996667 >= 500` fails.
6. User A's bid reverts, and the protocol loses the opportunity to execute a large bid at a favorable price.

**Recommendation**

Allow a range for `sellAmount` by accepting bids with different `minSellAmount` and `maxSellAmount` values via `Folio.bid()`, allow to fill as much as is available within the range at execution time.

Or, prevent griefing by either freezing `redeem()` during active auctions, or introducing a redemption fee during auctions to disincentivize such attacks.

---
### Example 2

**Auto Label:** Flawed state synchronization and improper validation lead to incorrect asset balances, misrouted fees, and unauthorized access, enabling attacks, DoS, and economic exploitation.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** High

## Description

Due to the lack of a check on the split amount inside `split`, a user can provide an arbitrary `_amount` to the function. Since `locked.amount` is stored as an `int128`, this can result in one of the split tokens having a negative value, while the other is set to the full `_amount` provided.

```solidity
    function split(
        uint _from,
        uint _amount
    ) external returns (uint _tokenId1, uint _tokenId2) {
        address msgSender = msg.sender;

        require(_isApprovedOrOwner(msgSender, _from));
        require(attachments[_from] == 0 && !voted[_from], "attached");
        require(_amount > 0, "Zero Split");

        // burn old NFT
        LockedBalance memory _locked = locked[_from];
        int128 value = _locked.amount;
        locked[_from] = LockedBalance(0, 0);
        _checkpoint(_from, _locked, LockedBalance(0, 0));
        _burn(_from);

        // set max lock on new NFTs
        _locked.end = block.timestamp + MAXTIME;

        int128 _splitAmount = int128(uint128(_amount));
        // @audit - underflow is possible
>>>     _locked.amount = value - _splitAmount; // already checks for underflow here in ^0.8.0
        _tokenId1 = _createSplitNFT(msgSender, _locked);

>>>     _locked.amount = _splitAmount;
        _tokenId2 = _createSplitNFT(msgSender, _locked);

        emit Split(
            _from,
            _tokenId1,
            _tokenId2,
            msgSender,
            uint(uint128(locked[_tokenId1].amount)),
            uint(uint128(_splitAmount)),
            _locked.end,
            block.timestamp
        );
    }
```

This can be abused by initially depositing a small amount of tokens, then calling `split` with an arbitrary amount to create a large `locked` data. Then can be withdrawn at the end of the lock period, and also grants an arbitrary amount of voting power and balance, which can be used to claim rewards from gauges and bribes.

PoC :

```solidity
    function testSplitVeKittenUnderflow() public {
        testDistributeVeKitten();

        vm.startPrank(user1);

        uint veKittenId = veKitten.tokenOfOwnerByIndex(user1, 0);
        (int128 lockedAmount, uint endTime) = veKitten.locked(veKittenId);

        (uint t1, uint t2) = veKitten.split(veKittenId, uint256(uint128(lockedAmount)) * 100);

        (int128 lockedAmountSplit, ) = veKitten.locked(t2);

        console.log("initial token to split locked amount :");
        console.log(lockedAmount);
        console.log("splitted token locked amount :");
        console.log(lockedAmountSplit);

        vm.stopPrank();
    }
```

Log output :

```shell
Logs:
  initial token to split locked amount :
  100000000000000000000000000
  splitted token locked amount :
  10000000000000000000000000000
```

## Recommendations

Validate that the provided `_amount` does not exceed the locked amount of the split token.

---
### Example 3

**Auto Label:** Flawed state synchronization and improper validation lead to incorrect asset balances, misrouted fees, and unauthorized access, enabling attacks, DoS, and economic exploitation.  

**Original Text Preview:**

## Severity

**Impact:** Medium, Because the cap raiser cannot update `maxTotalDeposits` and will wrongly update `maxPerDeposit` with `_newMaxTotalDeposits`.

**Likelihood:** Medium, Because it will happened every time cap raiser want to update TVL limits.

## Description

Current implementation of `setTVLLimits` is incorrectly assigns `_newMaxTotalDeposits` to `maxPerDeposit` instead of `maxTotalDeposits`.

```solidity
    function setTVLLimits(uint256 _newMaxPerDeposit, uint256 _newMaxTotalDeposits) external onlyCapRaiser() {
        require(_newMaxPerDeposit<=_newMaxTotalDeposits, "newMaxPerDeposit exceeds newMaxTotalDeposits");

        maxPerDeposit = _newMaxPerDeposit;
>>      maxPerDeposit = _newMaxTotalDeposits;

        emit MaxPerDepositUpdated(maxPerDeposit, _newMaxPerDeposit);
        emit MaxTotalDepositsUpdated(maxTotalDeposits, _newMaxTotalDeposits);
    }
```

## Recommendations

Assign `_newMaxTotalDeposits` to `maxTotalDeposits` instead of `maxPerDeposit`.

```diff
    function setTVLLimits(uint256 _newMaxPerDeposit, uint256 _newMaxTotalDeposits) external onlyCapRaiser() {
        require(_newMaxPerDeposit<=_newMaxTotalDeposits, "newMaxPerDeposit exceeds newMaxTotalDeposits");

        maxPerDeposit = _newMaxPerDeposit;
-       maxPerDeposit = _newMaxTotalDeposits;
+       maxTotalDeposits = _newMaxTotalDeposits;

        emit MaxPerDepositUpdated(maxPerDeposit, _newMaxPerDeposit);
        emit MaxTotalDepositsUpdated(maxTotalDeposits, _newMaxTotalDeposits);
    }
```

---
