# Cluster -1250

**Rank:** #8  
**Count:** 954  

## Label
Insufficient data validation and unchecked arithmetic lead to incorrect price calculations, enabling manipulation and loss of oracle integrity through unbounded growth or reliance on invalid historical data.

## Cluster Information
- **Total Findings:** 954

## Examples

### Example 1

**Auto Label:** Failure to validate oracle data freshness and integrity leads to stale, incorrect, or malicious pricing, causing financial mispricing, exploitation, and system instability.  

**Original Text Preview:**

Currently, the time interval for `shortTwap` is hardcoded to 3 seconds.

```solidity
    function shortTwap() public view returns (int56 twapTick) {
        uint32[] memory secondsAgo = new uint32[](2);
        secondsAgo[0] = uint32(3);
        secondsAgo[1] = 0;

        (int56[] memory tickCuml,) = IUniswapV3Pool(pool).observe(secondsAgo);
        twapTick = (tickCuml[1] - tickCuml[0]) / int32(3);
    }
```

If `twapInterval` for `twap`  is set to less than 3, `shortTwap` will not function as intended. Consider allowing the twap interval for `shortTwap` to be configurable.

---
### Example 2

**Auto Label:** Failure to validate oracle data freshness and integrity leads to stale, incorrect, or malicious pricing, causing financial mispricing, exploitation, and system instability.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

Inside `withdraw` operation of `StrategyPassiveManagerHyperswap` and `StrategyPassiveManagerKittenswap`, `_onlyCalmPeriods` will be checked only when `calmAction` is false and `lastDeposit` is equal to `block.timestamp`.

```solidity
    function withdraw(uint256 _amount0, uint256 _amount1) external {
        _onlyVault();

>>>     if (block.timestamp == lastDeposit && !calmAction) _onlyCalmPeriods();

        if (_amount0 > 0) IERC20Metadata(lpToken0).safeTransfer(vault, _amount0);
        if (_amount1 > 0) IERC20Metadata(lpToken1).safeTransfer(vault, _amount1);

        if (!_isPaused()) _addLiquidity();
    
        (uint256 bal0, uint256 bal1) = balances();
        calmAction = false;

        emit TVL(bal0, bal1);
    }
```

An attacker can exploit this by depositing dust amount, waiting for the next block, then performing a swap to manipulate the price. After that, they can call `withdraw` to trigger `_removeLiquidity` and `_addLiquidity`, causing the strategy to add liquidity using the manipulated price.

```solidity
    function _addLiquidity() private {
        _whenStrategyNotPaused();

        (uint256 bal0, uint256 bal1) = balancesOfThis();

        uint160 sqrtprice = sqrtPrice();
        uint128 liquidity = LiquidityAmounts.getLiquidityForAmounts(
            sqrtprice,
            TickMath.getSqrtRatioAtTick(positionMain.tickLower),
            TickMath.getSqrtRatioAtTick(positionMain.tickUpper),
            bal0,
            bal1
        );

        bool amountsOk = _checkAmounts(liquidity, positionMain.tickLower, positionMain.tickUpper);

>>>     if (liquidity > 0 && amountsOk) {
            minting = true;
            IKittenswapV3Pool(pool).mint(address(this), positionMain.tickLower, positionMain.tickUpper, liquidity, "Beefy Main");
        } else {
            if(!calmAction) _onlyCalmPeriods();
        }

        (bal0, bal1) = balancesOfThis();

        liquidity = LiquidityAmounts.getLiquidityForAmounts(
            sqrtprice,
            TickMath.getSqrtRatioAtTick(positionAlt.tickLower),
            TickMath.getSqrtRatioAtTick(positionAlt.tickUpper),
            bal0,
            bal1
        );

        // Flip minting to true and call the pool to mint the liquidity.
        if (liquidity > 0) {
            minting = true;
            IKittenswapV3Pool(pool).mint(address(this), positionAlt.tickLower, positionAlt.tickUpper, liquidity, "Beefy Alt");
        }
    }
```

This can happen because, inside `_addLiquidity`, the `_onlyCalmPeriods` check is skipped as long as the price remains within the `positionMain` ticks.

## Recommendations

Add `onlyCalmPeriods` modifier to `_addLiquidity`.

---
### Example 3

**Auto Label:** Failure to validate oracle data freshness and integrity leads to stale, incorrect, or malicious pricing, causing financial mispricing, exploitation, and system instability.  

**Original Text Preview:**

## Severity

**Impact:** High - Using stale prices leads to inaccurate calculations of total asset values and share prices.

**Likelihood:** Low - Price can be stale frequently if there is no update

## Description

The `StrategyV5Pyth` uses `pyth.getPriceUnsafe` for obtaining Pyth oracle price feeds to calculate the asset/input exchange rate.

        function assetExchangeRate(uint8 inputId) public view returns (uint256) {
            if (inputPythIds[inputId] == assetPythId)
                return weiPerShare; // == weiPerUnit of asset == 1:1
            PythStructs.Price memory inputPrice = pyth.getPriceUnsafe(inputPythIds[inputId]);
            PythStructs.Price memory assetPrice = pyth.getPriceUnsafe(assetPythId);
            ...
        }

However, from the Pyth documents, using the getPriceUnsafe can return stale price if the price is not updated.

        /// @notice Returns the price of a price feed without any sanity checks.
        /// @dev This function returns the most recent price update in this contract without any recency checks.
        /// This function is unsafe as the returned price update may be arbitrarily far in the past.
        ///
        /// Users of this function should check the `publishTime` in the price to ensure that the returned price is
        /// sufficiently recent for their application. If you are considering using this function, it may be
        /// safer / easier to use either `getPrice` or `getPriceNoOlderThan`.
        /// @return price - please read the documentation of PythStructs.Price to understand how to use this safely.
        function getPriceUnsafe(
            bytes32 id
        ) external view returns (PythStructs.Price memory price);

The `assetExchangeRate` function doesn't verify `Price.publishTime`, potentially leading to outdated exchange rates, incorrect investment calculations, and distorted total asset values.

## Recommendations

Using `pyth.updatePriceFeeds` for updating prices, followed by `pyth.getPrice` for retrieval.
Following the example in: https://github.com/pyth-network/pyth-sdk-solidity/blob/main/README.md#example-usage

---
