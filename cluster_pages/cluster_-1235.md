# Cluster -1235

**Rank:** #136  
**Count:** 104  

## Label
Inconsistent or stale oracle price usage leads to inaccurate collateral calculations, enabling exploitation through arbitrage, inflated ratios, and asymmetric incentives, ultimately undermining margin safety and economic disincentives.

## Cluster Information
- **Total Findings:** 104

## Examples

### Example 1

**Auto Label:** Common vulnerability type: **Price oracle misalignment enabling predictable arbitrage and collateral devaluation through unchecked price assumptions and lack of real-time validation.**  

**Original Text Preview:**

In BeamNode, users can choose to mint NFT via depositing native token, USDC, or BEAM token. The admin can set different base price for different deposit tokens. This will make sure that we have a similar deposit price with the different deposit tokens.

Although the admin can update these base prices according to the real-time token price, it's impossible to sync the token price all the time. If the native token or the BEAM token price changes a lot, users may mint NFT tokens quite cheaper than expected.

```solidity
    function getPrice(TokenType _tokenType) public view returns (uint256) {
        if (_tokenType == TokenType.NATIVE) {
            return nativeBasePrice;
        } else if (_tokenType == TokenType.USDC) {
            return usdcBasePrice;
        } else {
            return beamBasePrice;
        }
    }
```

It's suggested to use one chainlink oracle to calculate the real-time token price.

---
### Example 2

**Auto Label:** Precision errors in collateral calculations lead to incorrect asset valuations, undermining stability, risk assessment, and liquidation mechanisms across protocols.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Low

## Description

The `_redeemCollateralFromTrove()` function, used internally by `redeemCollateral()`, assumes all collateral tokens have 18 decimals, matching the debt token. However, when handling tokens with different precisions, incorrect calculations occur due to this formula:

```solidity
        singleRedemption.collateralLot = (singleRedemption.debtLot * DECIMAL_PRECISION) / _price;
```

Since `singleRedemption.debtLot`, `_price`, and `DECIMAL_PRECISION` are all 18-decimal values, this results in an invalid collateralLot for tokens with different decimals. This leads to:

- Underflows due to precision mismatches.
- Incorrect collateral balances, causing miscalculations in accounting.

Example issue in `newColl` calculation:

```solidity
        uint256 newColl = (t.coll) - singleRedemption.collateralLot;
```

Additionally, `_updateBaseRateFromRedemption()` is affected, as `_CollateralDrawn` should match the collateral’s precision, while \_price and `_totalDebtSupply` remain in 1e18 precision.

## Recommendations

Ensure all calculations are normalized to a consistent precision(i.e. 18 decimals).

---
### Example 3

**Auto Label:** Inaccurate asset value calculations due to flawed balance tracking or improper weight/fee handling, enabling unjust liquidations, fee theft, or denial-of-service through manipulated risk exposure.  

**Original Text Preview:**

**Impact**

The impact from the current formatting is that manual review takes longer, and as such will be more expensive and error prone

**Specific Code Smells**

- Not using curly braces for `if / else`
- Using `ii` in favour of mathematical notation `n, l, m` `i, j, k`, `x, y, z`
- Breaking `CEI`
- Ping ponging of external calls
- Subtle changes against Liquity's version


**Breaking CEI**

`openTrove` shows an example of a seemingly innocuous way to break CEI

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/BorrowerOperations.sol#L199-L224

```solidity
    vars.arrayIndex = contractsCache.troveManager.addTroveOwnerToArray(borrower);
    /// @audit move the transfer of coll to the bottom to avoid CEI issues
    // Move the coll to the active pool
    for (uint i = 0; i < vars.colls.length; i++) {
      TokenAmount memory collTokenAmount = vars.colls[i]; /// @audit CEI / Reentrancy
      _poolAddColl(
        borrower,
        contractsCache.storagePool,
        collTokenAmount.tokenAddress,
        collTokenAmount.amount,
        PoolType.Active
      );
    }
    /// @audit Where is the trove? 
    // Move the stable coin gas compensation to the Gas Pool
    contractsCache.storagePool.addValue(
      address(stableCoinAmount.debtToken),
      false,
      PoolType.GasCompensation,
      stableCoinAmount.netDebt
    ); /// @audit Not minting principal on open
    stableCoinAmount.debtToken.mint(address(contractsCache.storagePool), stableCoinAmount.netDebt);

    emit TroveCreated(borrower, _colls);
  }

```


With `_poolAddColl` looking as follows

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/BorrowerOperations.sol#L840-L851

```solidity

  function _poolAddColl(
    address _borrower,
    IStoragePool _pool,
    address _collAddress,
    uint _amount,
    PoolType _poolType
  ) internal {
    _pool.addValue(_collAddress, true, _poolType, _amount);
    IERC20(_collAddress).transferFrom(_borrower, address(_pool), _amount); /// @audit FOT / SafeTransfer
  }

```

This would make it so that the system is recording the increase in collateral, but not the increase in debt, which would allow to drag the TCR below the Recovery Mode threshold, and liquidate all Troves that have a ICR < CCR

Liquity like systems tend to blur the idea of an external call as they effectively rely on external calls for accounting, but those can be viewed as trusted external calls

Whereas moving tokens should be viewed as an untrusted external call under the vast majority of circumnstances

You should refactor not to break CEI as it will:
- Ensure you cannot introduce exploits tied to ordering of external calls
- Make future review cheaper as CEI concerns will not take a considerable amount of time for review

---
