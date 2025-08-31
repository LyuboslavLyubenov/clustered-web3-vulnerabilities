# Cluster -1454

**Rank:** #437  
**Count:** 9  

## Label
Timing and pricing misalignments enable attackers to exploit arithmetic errors or dynamic price fluctuations, leading to inflated collateral values, unprofitable mints, or arbitrage profits through incorrect value calculations.

## Cluster Information
- **Total Findings:** 9

## Examples

### Example 1

**Auto Label:** Flawed collateral or pricing calculations enable attackers to manipulate debt issuance or redemption rates, bypassing risk controls and extracting value through inaccurate state or price assumptions.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-12-plaza-finance-judging/issues/896 

## Found by 
056Security, 0xc0ffEE, KupiaSec, bigbear1229, future, novaman33, t0x1c, zraxx

### Summary

Buying `LeverageToken` increases TVL, which in turn raises the `collateralLevel`.

When redeeming `BondToken`, the redemption amount is determined by the `collateralLevel`. The calculation of the redemption amount varies depending on whether the `collateralLevel` is above or below `120%`.

Therefore, `BondToken` redeemers can acquire more underlying assets by manipulating the `collateralLevel` from `< 120%` to `> 120%` through purchasing `LeverageToken`, ultimately resulting in a profit.

### Root Cause

The [getRedeemAmount()](https://github.com/sherlock-audit/2024-12-plaza-finance/tree/main/plaza-evm/src/Pool.sol#L511-L518) function calculates the `redeemRate` based on whether the `collateralLevel` is above or below `120%`.

When `collateralLevel < 120%`, `80%` of TVL is allocated for `BondToken` holders. In contrast, when `collateralLevel > 120%`, the price of `BondToken` is fixed at `100`.

This vulnerability provides malicious users with an opportunity to manipulate the `collateralLevel` by purchasing `LeverageToken`, allowing them to redeem their `BondToken`s at a higher rate.

```solidity
      function getRedeemAmount(
        ...
        
        uint256 collateralLevel;
        if (tokenType == TokenType.BOND) {
498       collateralLevel = ((tvl - (depositAmount * BOND_TARGET_PRICE)) * PRECISION) / ((bondSupply - depositAmount) * BOND_TARGET_PRICE);
        } else {
          multiplier = POINT_TWO;
          assetSupply = levSupply;
502       collateralLevel = (tvl * PRECISION) / (bondSupply * BOND_TARGET_PRICE);
        ...
        
        uint256 redeemRate;
511     if (collateralLevel <= COLLATERAL_THRESHOLD) {
          redeemRate = ((tvl * multiplier) / assetSupply);
513     } else if (tokenType == TokenType.LEVERAGE) {
          redeemRate = ((tvl - (bondSupply * BOND_TARGET_PRICE)) / assetSupply) * PRECISION;
515     } else {
          redeemRate = BOND_TARGET_PRICE * PRECISION;
        }
        
        ...
      }
```

### Internal pre-conditions

### External pre-conditions

### Attack Path

Let's consider the following scenario:

- Current State of the Pool:
    - `levSupply`: 100
    - `bondSupply`: 100
    - `TVL`: $11000
- Bob wants to redeem `50 BondToken`. Expected Values:
    - `collaterlLevel`: (11000 - 100 * 50) / (100 - 50) = 120% (see line 498)
    - Price of `BondToken`: 11000 * 0.8 / 100 = 88 (see the case at line 511)
    - Price of `LeverageToken`: 11000 * 0.2 / 100 = 22 (see the case at line 511)

As a result, Bob can only redeem `50 * 88 = 4400`.

However, Bob manipulates `collateralLevel`.

1. Bob buys `10 LeverageToken` by using `$220`:
    - `levSupply`: 100 + 10 = 110
    - `bondSupply`: 100
    - `TVL`: 11000 + 220 = 11220
2. Bob then sells `50 BondToken`:
    - `collaterlLevel`: (11220 - 100 * 50) / (100 - 50) = 124.4% (see line 498)
    - price of `BondToken`: 100 (see the case at line 515)
    
    Bob receives `100 * 50 = 5000`.
    
    - `TVL`: 11220 - 5000 = 6220
    - `bondSupply`: 100 - 50 = 50
3. Bob sells back `10 LeverageToken`.
    - `collaterlLevel`: 6220 / 50 = 124.4% (see line 502)
    - Price of `LeverageToken`: (6220 - 100 * 50) / 110 = 11 (see the case at line 513)
    - Bob receives `10 * 11 = 110`.

As you can see, Bob was initially able to redeem only `$4400`. However, by manipulating `collateralLevel`, he can increase his redemption to `-220 + 5000 + 110 = 4890`. Thus, he can profit by `4890 - 4400 = 490`.

### Impact

`BondToken` redeemers can obtain more than they are entitled to by manipulating the `collateralLevel` through purchasing `LeverageToken`.

### PoC

### Mitigation

The current price mechanism should be improved.

## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/Convexity-Research/plaza-evm/pull/155

---
### Example 2

**Auto Label:** Flawed collateral or pricing calculations enable attackers to manipulate debt issuance or redemption rates, bypassing risk controls and extracting value through inaccurate state or price assumptions.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-12-plaza-finance-judging/issues/341 

## Found by 
0xadrii, KupiaSec, farman1094, future, novaman33


### Summary
An attacker can purchase bondETH at a low price and sell it at a higher price.

### Root Cause
When selling `bondETH`, the `estimated collateralLevel` is utilized instead of the `current collateralLevel`.
By exploiting this vulnerability, an attacker can purchase `bondETH` at various prices and sell it for the maximum price of $100.

https://github.com/sherlock-audit/2024-12-plaza-finance/tree/main/plaza-evm/src/Pool.sol#L498
```solidity
    function getRedeemAmount(
        ...
    ) public pure returns(uint256) {
        ...
        uint256 tvl = (ethPrice * poolReserves).toBaseUnit(oracleDecimals);
        uint256 assetSupply = bondSupply;
        uint256 multiplier = POINT_EIGHT;

        // Calculate the collateral level based on the token type
        uint256 collateralLevel;
        if (tokenType == TokenType.BOND) {
498:        collateralLevel = ((tvl - (depositAmount * BOND_TARGET_PRICE)) * PRECISION) / ((bondSupply - depositAmount) * BOND_TARGET_PRICE);
        } else {
            ...
        }
        
        // Calculate the redeem rate based on the collateral level and token type
        uint256 redeemRate;
        if (collateralLevel <= COLLATERAL_THRESHOLD) {
            redeemRate = ((tvl * multiplier) / assetSupply);
        } else if (tokenType == TokenType.LEVERAGE) {
            redeemRate = ((tvl - (bondSupply * BOND_TARGET_PRICE)) / assetSupply) * PRECISION;
        } else {
            redeemRate = BOND_TARGET_PRICE * PRECISION;
        }

        if (marketRate != 0 && marketRate < redeemRate) {
            redeemRate = marketRate;
        }
        
        // Calculate and return the final redeem amount
        return ((depositAmount * redeemRate).fromBaseUnit(oracleDecimals) / ethPrice) / PRECISION;
    }
```

### Internal pre-conditions
`collateralLevel > 1.2`

### External pre-conditions
N/A

### Attack Path
The attacker purchases bondETH until the collateralLevel is less than 1.2 at various prices and then sells it all at the maximum price($100).

### PoC
The price calculation formula is as follows:
- When purchasing bondETH:
`tvl = (ethPrice * poolReserve)`, `collateralLevel = tvl / (bondSupply * 100)`.
If `collateralLevel <= 1.2`, `creationRate = tvl * 0.8 / bondSupply`.
If `collateralLevel > 1.2`,  `creationRate = 100`.
- When selling bondETH:
`tvl = (ethPrice * poolReserve)`, `collateralLevel = (tvl - bondToSell * 100) / ((bondSupply - bondToSell) * 100)`.
If `collateralLevel <= 1.2`, `redeemRate = tvl * 0.8 / bondSupply`.
If `collateralLevel > 1.2`,  `redeemRate = 100`.

Assuming: `poolReserve = 120 ETH`, `bondSupply = 3000 bondETH`, `levSupply = 200 levETH`, `ETH Price = $3075`
- When Alice buys bondETH for 30 ETH:
    `tvl = 3075 * 120 = 369000`, `collateralLevel = 369000 / (3000 * 100) = 1.23`, `creationRate = 100`.
    `minted = 30 * 3075 / 100 = 922.5 bondETH`.
    `poolReserve = 150 ETH`, `bondSupply = 3922.5`, `Alice's bondEth amount = 922.5 bondETH`.
- When Alice buys bondETH another 30 ETH:
    `tvl = 3075 * 150 = 461250`, `collateralLevel = 461250 / (3922.5 * 100) ~= 1.176 < 1.2`, `creationRate = 461250 * 0.8 / 3922.5 ~= 94.07`
    `minted = 30 * 3075 / (461250 * 0.8 / 3922.5) = 980.625`
    `poolReserve = 180 ETH`, `bondSupply = 4903.125`. `Alice's bondEth amount = 1903.125 bondETH`.
- When Alice sells all of her bondETH:
    `tvl = 3075 * 180 = 553500`, `collateralLevel = (553500 - 1903.125 * 100) / (3000 * 100) = 363187.5 / 300,000 = 1.210625 > 1.2`
    `redeemRate = 100`, `receivedAmount = 1903.125 * 100 / 3075 ~= 61.89 ETH`.
Thus, Alice extracts approximately 1.89 ETH from this market. 
When Alice first buys at the price(creationRate) of $100, the market price (marketRate) is also nearly $100, resulting in no significant impact from the market price(Even if the decimal of `marketRate` is correct).

Attacker can extract ETH until `collateralLevel` reaches `1.2`.
This amount is `(ethPrice * poolReserve - 120 * bondSupply) ($)`.
Even if `collateralLevel < 1.2`, bondETH owners could sell their bondETH and then extract ETH from this market.

### Impact
An attacker could extract significant amounts of ETH from this market.

### Mitigation
```diff
    function getRedeemAmount(
        ...
    ) public pure returns(uint256) {
        ...
        // Calculate the collateral level based on the token type
        uint256 collateralLevel;
        if (tokenType == TokenType.BOND) {
-498:        collateralLevel = ((tvl - (depositAmount * BOND_TARGET_PRICE)) * PRECISION) / ((bondSupply - depositAmount) * BOND_TARGET_PRICE);
+498:        collateralLevel = (tvl * PRECISION) / (bondSupply * BOND_TARGET_PRICE);
        } else {
            ...
        }
        
        // Calculate the redeem rate based on the collateral level and token type
        uint256 redeemRate;
        if (collateralLevel <= COLLATERAL_THRESHOLD) {
            redeemRate = ((tvl * multiplier) / assetSupply);
        } else if (tokenType == TokenType.LEVERAGE) {
            redeemRate = ((tvl - (bondSupply * BOND_TARGET_PRICE)) / assetSupply) * PRECISION;
        } else {
            redeemRate = BOND_TARGET_PRICE * PRECISION;
        }

        if (marketRate != 0 && marketRate < redeemRate) {
            redeemRate = marketRate;
        }
        
        // Calculate and return the final redeem amount
        return ((depositAmount * redeemRate).fromBaseUnit(oracleDecimals) / ethPrice) / PRECISION;
    }
```    

### Test Code
https://github.com/sherlock-audit/2024-12-plaza-finance/tree/main/plaza-evm/test/Pool.t.sol#L1156
Changed linked function to following code.

```solidity
  function testCreateRedeemWithFees() public {
    vm.startPrank(governance);

    // Create a pool with 2% fee
    params.fee = 20000; // 2% fee (1000000 precision)
    params.feeBeneficiary = address(0x942);

    // Mint and approve reserve tokens
    Token rToken = Token(params.reserveToken);
    rToken.mint(governance, 120 ether);
    rToken.approve(address(poolFactory), 120 ether);

    Pool pool = Pool(poolFactory.createPool(params, 120 ether, 3000 ether, 200 ether, "", "", "", "", false));
    vm.stopPrank();

    // User creates leverage tokens
    vm.startPrank(user);
    
    rToken.mint(user, 60 ether);
    mockPriceFeed.setMockPrice(3075 * int256(CHAINLINK_DECIMAL_PRECISION), uint8(CHAINLINK_DECIMAL));

    uint256 usedEth = 60 ether;
    uint256 receivedEth = 0;
    uint256 buyTime = 2;
    uint256 sellTime = 1;
    rToken.approve(address(pool), usedEth);
    uint256 bondAmount = 0;

    console2.log("Before Balance:", rToken.balanceOf(user));
    assertEq(rToken.balanceOf(user), 60 ether);
    for (uint256 i = 0; i < buyTime; i++) {
      bondAmount += pool.create(Pool.TokenType.BOND, usedEth / buyTime, 0);
    }
    pool.bondToken().approve(address(pool), bondAmount);
    for (uint256 i = 0; i < sellTime; i++) {
      receivedEth += pool.redeem(Pool.TokenType.BOND, bondAmount / sellTime, 0);
    }
    console2.log(" After Balance:",rToken.balanceOf(user));
    assertLt(rToken.balanceOf(user), 60 ether);
    
    vm.stopPrank();

    // Reset state
    rToken.burn(user, rToken.balanceOf(user));
    rToken.burn(address(pool), rToken.balanceOf(address(pool)));
  }
```
forge test --match-test "testCreateRedeemWithFees" -vv

Result:
>[FAIL: assertion failed: 61890244154579369433 >= 60000000000000000000] testCreateRedeemWithFees() (gas: 2190059)
>Logs:
>  Before Balance: 60000000000000000000
>   After Balance: 61890244154579369433





## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/Convexity-Research/plaza-evm/pull/155

---
### Example 3

**Auto Label:** Flawed collateral or pricing calculations enable attackers to manipulate debt issuance or redemption rates, bypassing risk controls and extracting value through inaccurate state or price assumptions.  

**Original Text Preview:**

##### Description

The BondSteerOracle contract implements a flawed methodology for calculating the price of Steer Protocol LP tokens. The `_currentPrice` function uses spot reserves to determine the LP token value, which is susceptible to manipulation:

```
function _currentPrice(ERC20 quoteToken_, ERC20 payoutToken_) internal view override returns (uint256) {
    SteerParams memory params = priceFeedParams[quoteToken_][payoutToken_];
    (uint256 reserve0, uint256 reserve1) = params.steerVault.getTotalAmounts();
    uint256 totalSupply = params.steerVault.totalSupply();

    uint256 lpPrice;
    if (params.priceInToken0) {
        lpPrice = (reserve0 * 2 * PRECISION) / (totalSupply * (10 ** params.token0Decimals));
    } else {
        lpPrice = (reserve1 * 2 * PRECISION) / (totalSupply * (10 ** params.token1Decimals));
    }
    return PRECISION / lpPrice;
}
```

  

This implementation is vulnerable because:

1. It relies on spot reserves (`getTotalAmounts()`), which are easily manipulable through large trades or flash loans.
2. The calculation does not account for potential imbalances in the pool or market conditions.
3. There are no safeguards against extreme price movements or manipulation attempts.

##### Proof of Concept

This test can be found in BondFixedTermSteerOFDA.test.ts (with a modified MockSteerPool) :

```
it("should demonstrate price manipulation through reserve changes", async () => {
        // Configure oracle
        const encodedSteerConfig = ethers.utils.defaultAbiCoder.encode(
            ["tuple(address, bool)"], 
            [[steerToken.address, false]]
        );
        await oracle.setPair(steerToken.address, payoutToken.address, true, encodedSteerConfig);

        // Get initial LP token price
        const initialPrice = await oracle["currentPrice(address,address)"](steerToken.address, payoutToken.address);
        console.log("Initial LP token price:", initialPrice.toString());

        // Attacker manipulates reserves by adding large amount of liquidity with imbalanced reserves
        // This creates a price discrepancy
        await steerToken.setToken0Amount(ethers.utils.parseEther("10000")); // Large amount of token0
        await steerToken.setToken1Amount(ethers.utils.parseEther("100")); // Keep token1 small
        await steerToken._setTotalSupply(ethers.utils.parseEther("1000")); // Increase total supply proportionally

        // Get manipulated price
        const manipulatedPrice = await oracle["currentPrice(address,address)"](steerToken.address, payoutToken.address);
        console.log("Manipulated LP token price:", manipulatedPrice.toString());

        // Create bond market with manipulated price
        const encodedParams = ethers.utils.defaultAbiCoder.encode(
            ["tuple(address, address, address, address, uint48, uint48, bool, uint256, uint48, uint48, uint48, uint48)"],
            [
                [
                    payoutToken.address,
                    steerToken.address,
                    ethers.constants.AddressZero,
                    oracle.address,
                    "20000", // baseDiscount
                    "50000", // maxDiscountFromCurrent 
                    false,   // capacityInQuote
                    ethers.utils.parseEther("10000"), // capacity
                    "360000", // depositInterval
                    vestingLength,
                    "0", // start
                    bondDuration,
                ]
            ]
        );

        await auctioner.createMarket(encodedParams);
        const bondMarketPrice = await auctioner.marketPrice(0);
        console.log("Bond market price:", bondMarketPrice.toString());

        // Verify manipulation occurred
        expect(manipulatedPrice).to.be.gt(initialPrice);

        // Attacker can now remove the extra liquidity
        await steerToken.setToken0Amount(ethers.utils.parseEther("100"));
        await steerToken.setToken1Amount(ethers.utils.parseEther("100"));
        await steerToken._setTotalSupply(ethers.utils.parseEther("100"));

        const finalPrice = await oracle["currentPrice(address,address)"](steerToken.address, payoutToken.address);
        console.log("Final LP token price:", finalPrice.toString());

        // Price should have been manipulated up and then back down
        expect(finalPrice).to.be.lt(manipulatedPrice);
        
        // Calculate manipulation percentage
        const manipulationPercent = manipulatedPrice.sub(initialPrice).mul(100).div(initialPrice);
        console.log("Price manipulation percentage:", manipulationPercent.toString(), "%");
    });
```

Here is the result :

![PriceManipulation.png](https://halbornmainframe.com/proxy/audits/images/672a95c47c64a30584f29721)

##### BVSS

[AO:A/AC:L/AX:M/C:N/I:N/A:N/D:C/Y:L/R:N/S:U (7.1)](/bvss?q=AO:A/AC:L/AX:M/C:N/I:N/A:N/D:C/Y:L/R:N/S:U)

##### Recommendation

It is recommended to implement Alpha Finance's formula for pricing LP tokens:

`2 sqrt(p0 p1) * sqrt(k) / L`

Where:

* `p0` and `p1` are time-weighted average prices (TWAPs) of the underlying assets
* `k` is the constant product invariant (`k = r0 * r1`)
* `L` is the total supply of LP tokens

  

This approach calculates the LP token price based on TWAPs and the invariant k, rather than using manipulable spot reserves. It provides resistance against flash loan attacks and other forms of price manipulation.

##### Remediation

**SOLVED:** The **Lucid Labs team** implemented a lot of changes, including the recommended solution and an oracle implementation for the tokens prices of the LP Vault.

##### Remediation Hash

<https://github.com/LucidLabsFi/demos-contracts-v1/commit/851ca93f35e79cb38b96eafbac9441997e5ae7be>

##### References

[LucidLabsFi/demos-contracts-v1/contracts/modules/bonding/BondSteerOracle.sol#L55](https://github.com/LucidLabsFi/demos-contracts-v1/blob/main/contracts/modules/bonding/BondSteerOracle.sol#L55)

---
