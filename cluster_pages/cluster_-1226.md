# Cluster -1226

**Rank:** #408  
**Count:** 11  

## Label
Misconfigured pricing logic and flawed state validation enable attackers or users to manipulate prices, bypass controls, or face denial of service—undermining accuracy, transparency, and usability in token pricing mechanisms.

## Cluster Information
- **Total Findings:** 11

## Examples

### Example 1

**Auto Label:** Misconfigured pricing logic and flawed state validation enable attackers or users to manipulate prices, bypass controls, or face denial of service—undermining accuracy, transparency, and usability in token pricing mechanisms.  

**Original Text Preview:**

When users want to sell their tokens by calling `sellPoints()` the code calculates the value and also updates the points price:

```solidity
  function sellPoints(EEmpire _empire, uint256 _points) public _onlyNotGameOver {
    uint256 pointSaleValue = LibPrice.getPointSaleValue(_empire, _points);

    // require that the pot has enough ETH to send
    require(pointSaleValue <= Balances.get(EMPIRES_NAMESPACE_ID), "[OverrideSystem] Insufficient funds for point sale");

    // remove points from player and empire's issued points count
    LibPoint.removePoints(_empire, playerId, _points);

    // set the new empire point price
    LibPrice.sellEmpirePointPriceDown(_empire, _points);
--snip
```

The issue is that during value calculation or price update code won't allow selling tokens if the price reaches the minimum price:

```solidity
    require(
      currentPointPrice >= config.minPointPrice + pointPriceDecrease * wholePoints,
      "[LibPrice] Selling points beyond minimum price"
    );
```

The price of points increases with each buy and decreases the same amount with each sale in most situations users would able to sell all their points. But the point's price decreases overtime when admin calls `turnEmpirePointPriceDown()` so as a result point's price would reach the minimum price when there are some points left and some users won't be able to sell their points even if so contract has enough balance.

## Recommendations

Instead of reverting when the price reaches the minimum limit, the code should allow selling for the minimum price and doesn't change the price.

---
### Example 2

**Auto Label:** Lack of price validation enables attackers to manipulate token prices or exchange rates, allowing them to extract assets or inflate supply through front-running and MEV exploitation.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Low

## Description

When buying a coin through `buy()` in GroupcoinFactory.sol, the buyer will attach some `msg.value` to the function in order to buy a certain number of coins. In `buy()`, there is no slippage protection, so a malicious user can buy

For example,

- Alice wants to buy 1000 Acoin. She is the first buyer and she calculates that she needs about 3434501 wei of ETH to buy (amount calculated from the `getPrice()` function. Alice knows that the protocol will refund any excess ETH that she sends in, so she simply passes in 1e18 worth of ETH.
- Bob sees that transaction and frontruns her, by buying 1,000,000 Acoin for about 3429360425241769 wei. Now, Alice's transaction goes through, and `totalSupply()` of Acoin becomes 1 million instead of 0.
- Alice 1000 Acoins is now 10298367632031 wei of ETH, and the transfer goes through because she attached 1e18 worth of ETH.

## Recommendations

To protect the buyers, add another parameter `maxPrice` to make sure that the buyer is willing to pay x amount of tokens until it reaches the max price. In Alice's case, her max price would be 4000000 wei, so her transaction will be reverted if the coin is any price above that.

Sample example:

```
    function buy(uint256 tokenId, uint256 amount, uint256 maxPrice) public payable {
        ...
        if (msg.value < price + protocolFeeTaken) {
            revert NotEnough();
        }
        if(maxPrice != 0){
        require(maxPrice > price + protocolFeeTaken, "Price too expensive")
        }
```

If the maxPrice is not set, then it is assumed that the buyer accepts that there is no slippage protection.

---
### Example 3

**Auto Label:** Misconfigured pricing logic and flawed state validation enable attackers or users to manipulate prices, bypass controls, or face denial of service—undermining accuracy, transparency, and usability in token pricing mechanisms.  

**Original Text Preview:**

**Severity**

**Impact:** High

**Likelihood:** High

**Description**

ToyBox uses `customSale()` and `customSaleWithPermit()` for non-primary sales, and these functions should not use data from the primary sales.
But customSaleWithPermit() uses the price for the primary sale, which likely is not the intended behavior.

```
    function customSaleWithPermit(uint256 _amount, PermitSignature calldata _permitSignature, bytes32[] calldata _proof)
        external
        nonReentrant
        customSaleChecks(_permitSignature.owner, _permitSignature.token, _amount, _proof)
    {
        ...
        _collectWithPermit(
            _amount, getFullCustomPrice(price, _permitSignature.token), saleStruct.referenceToken, _permitSignature
        );
        ...
    }
```

As a result, users signing approvals via permit will receive a different price.

**Recommendations**

Replace `price` with `saleStruct.price`, as in `customSale()`.

---
