# Cluster -1452

**Rank:** #196  
**Count:** 58  

## Label
Inconsistent or flawed price calculations due to incorrect rounding, invalid ratio checks, and improper validation of price inputs lead to erroneous asset valuations, compromised safety, and cascading financial errors.

## Cluster Information
- **Total Findings:** 58

## Examples

### Example 1

**Auto Label:** Inconsistent or flawed price calculations due to incorrect rounding, invalid ratio checks, and improper validation of price inputs lead to erroneous asset valuations, compromised safety, and cascading financial errors.  

**Original Text Preview:**

## Severity

High Risk

## Description

The `click()` function in the `Multicall` contract does not verify that the received `msg.value` matches the calculated `price`, which is determined by calling `IQueuePlugin(plugin).getPrice() * amount`. This creates a vulnerability where users may send excessive ETH that gets locked in the contract when the plugin owner decreases the `entryFee`.

```solidity
function click(uint256 tokenId, uint256 amount, string calldata message) external payable returns (uint256 mintAmount) {
    uint256 price = IQueuePlugin(plugin).getPrice() * amount;
    IWBERA(base).deposit{value: price}();
    // No validation that msg.value == price
    IERC20(base).safeApprove(plugin, 0);
    IERC20(base).safeApprove(plugin, price);

    for (uint256 i = 0; i < amount; i++) {
        mintAmount += IQueuePlugin(plugin).click(tokenId, message);
    }
}
```

The plugin owner can modify the `entryFee` through the `setEntryFee()` function, which directly impacts the price calculation:

```solidity
function getPrice() external view returns (uint256) {
    return entryFee;
}
```

```solidity
function setEntryFee(uint256 _entryFee) external onlyOwner {
    entryFee = _entryFee;
    emit Plugin__EntryFeeSet(_entryFee);
}
```

## Location of Affected Code

File: [Multicall.sol](https://github.com/Heesho/bull-ish/blob/3b48a94280e1ddcc25f486735508a50c70a21741/contracts/Multicall.sol)

## Impact

Users may send excessive ETH that gets locked in the contract when the plugin owner decreases the `entryFee`.

## Proof of Concept

1. A user calculates the required payment based on the current `entryFee` (e.g., 1 ETH per click) and amount (e.g., 10 clicks), resulting in 10 ETH.
2. The user sends a transaction with 10 ETH as `msg.value` to call the `click` function.
3. Before the user's transaction is processed, the plugin owner reduces the `entryFee` from 1 ETH to 0.5 ETH.
4. When the user's transaction executes, the function calculates `price = 0.5 ETH * 10 = 5 ETH`.
5. Only 5 ETH is used in the `deposit` call, while the remaining 5 ETH is stuck in the contract with no mechanism to retrieve it.

## Recommendation

Check the relationship between `msg.value` and `price`. If there’s extra, return it to the user; if it’s not enough, revert.

## Team Response

Fixed.

---
### Example 2

**Auto Label:** Inconsistent or flawed price calculations due to incorrect rounding, invalid ratio checks, and improper validation of price inputs lead to erroneous asset valuations, compromised safety, and cascading financial errors.  

**Original Text Preview:**

## InitOrder Instruction

In the `InitOrder` instruction, the `auction_mode` parameter is intended to control the order’s behavior by determining whether it enters an auction phase or proceeds directly to fulfillment. Specifically, if `auction_mode = 1`, the order enters the auction phase, and if `auction_mode = 2`, the order goes directly to fulfillment. 

However, currently, there is no validation to ensure that `auction_mode` is strictly either 1 or 2. If an invalid value is passed, the order will fail to enter either auction or fulfillment, leading to cancellation and refunding of the order.

## Remediation

Ensure `auction_mode` is explicitly validated at the start of the `init_order`, to prevent processing any invalid values, returning an error instead.

## Patch

Acknowledged by the developers and intended behaviour.

---
### Example 3

**Auto Label:** Inconsistent or flawed price calculations due to incorrect rounding, invalid ratio checks, and improper validation of price inputs lead to erroneous asset valuations, compromised safety, and cascading financial errors.  

**Original Text Preview:**

##### Description

The `minBidIncrementPercentage`parameter in the **BeraAuctionHouse** contract defines the minimum percentage increase for new bids. Although the `setMinBidIncrementPercentage`function includes a check to prevent setting this value to zero, there is no validation to ensure the parameter is within a reasonable range. If the value is set too low, it could lead to auction inefficiencies by encouraging small incremental bids, which may not be economically meaningful. Conversely, if the value is set too high, it could discourage participation by requiring bidders to place unreasonably large increments.

  

Additionally, the parameter is mutable, meaning it can be updated at any time. Without proper safeguards, this could lead to unexpected behavior if the value is set to an impractical level during the contract's operation.

  

Code Location
-------------

The `constructor` does not validate whether `minBidIncrementPercentage` falls within a reasonable range:

```
    constructor(
        BaseRegistrar base_,
        BeraDefaultResolver resolver_,
        IWETH weth_,
        uint256 auctionDuration_,
        uint256 registrationDuration_,
        uint192 reservePrice_,
        uint56 timeBuffer_,
        uint8 minBidIncrementPercentage_,
        address paymentReceiver_
    ) Ownable(msg.sender) {
        base = base_;
        resolver = resolver_;
        weth = weth_;
        auctionDuration = auctionDuration_;
        registrationDuration = registrationDuration_;
        paymentReceiver = paymentReceiver_;

        _pause();

        reservePrice = reservePrice_;
        timeBuffer = timeBuffer_;
        minBidIncrementPercentage = minBidIncrementPercentage_;
    }
```

  

The `setMinBidIncrementPercentage` function does not validate whether `minBidIncrementPercentage` falls within a reasonable range:

```
    function setMinBidIncrementPercentage(uint8 _minBidIncrementPercentage) external override onlyOwner {
        if (_minBidIncrementPercentage == 0) {
            revert MinBidIncrementPercentageIsZero();
        }

        minBidIncrementPercentage = _minBidIncrementPercentage;

        emit AuctionMinBidIncrementPercentageUpdated(_minBidIncrementPercentage);
    }
```

##### BVSS

[AO:A/AC:L/AX:M/R:N/S:U/C:N/A:N/I:M/D:M/Y:N (4.2)](/bvss?q=AO:A/AC:L/AX:M/R:N/S:U/C:N/A:N/I:M/D:M/Y:N)

##### Recommendation

Enforce an acceptable range for `minBidIncrementPercentage`in both the `constructor`and the `setMinBidIncrementPercentage`function. A range between 5% and 10% is common in most auction systems.

##### Remediation

**RISK ACCEPTED:** The **Beranames team** updated the constructor code to validate that `minBidIncrementPercentage_` is not zero. This change was introduced in commit [e0e6ff2](https://github.com/Beranames/beranames-contracts-v2/pull/91/commits/e0e6ff260934a31241bad54e2952c913c7bab83c). However, they chose to accept the remaining risk as they preferred not to impose an upper limit on `minBidIncrementPercentage_`.

---
