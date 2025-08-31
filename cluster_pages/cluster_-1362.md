# Cluster -1362

**Rank:** #283  
**Count:** 28  

## Label
Lack of access control and input validation enabling unauthorized modifications or bypasses of critical security checks, leading to fund exposure, protocol manipulation, or unauthorized operations.

## Cluster Information
- **Total Findings:** 28

## Examples

### Example 1

**Auto Label:** **Unauthorized external contract execution leading to fund theft via unvalidated inputs and unchecked calls.**  

**Original Text Preview:**

**Med - Missing Sweeps here**

https://github.com/liquity/bold/blob/3f190ec5d63fa26a64fa5edc9404b92e9e053e03/contracts/src/Zappers/GasCompZapper.sol#L143-L174

```solidity
    function adjustTroveWithRawETH(
        uint256 _troveId,
        uint256 _collChange,
        bool _isCollIncrease,
        uint256 _boldChange,
        bool _isDebtIncrease,
        uint256 _maxUpfrontFee
    ) external {
        address receiver = _adjustTrovePre(_troveId, _collChange, _isCollIncrease, _boldChange, _isDebtIncrease);
        borrowerOperations.adjustTrove(
            _troveId, _collChange, _isCollIncrease, _boldChange, _isDebtIncrease, _maxUpfrontFee
        );
        _adjustTrovePost(_collChange, _isCollIncrease, _boldChange, _isDebtIncrease, receiver);
    }

    function adjustUnredeemableTroveWithRawETH(
        uint256 _troveId,
        uint256 _collChange,
        bool _isCollIncrease,
        uint256 _boldChange,
        bool _isDebtIncrease,
        uint256 _upperHint,
        uint256 _lowerHint,
        uint256 _maxUpfrontFee
    ) external {
        address receiver = _adjustTrovePre(_troveId, _collChange, _isCollIncrease, _boldChange, _isDebtIncrease);
        borrowerOperations.adjustUnredeemableTrove(
            _troveId, _collChange, _isCollIncrease, _boldChange, _isDebtIncrease, _upperHint, _lowerHint, _maxUpfrontFee
        );
        _adjustTrovePost(_collChange, _isCollIncrease, _boldChange, _isDebtIncrease, receiver);
    }

```

As discussed in the original finding, one way to leave dust is to try repaying more than necessary

This will result in a transfer in of boldToken
The unused boldToken will be stuck in the contract


**Undefined - Arbitrary input is dangerous vs using `msg.sender`**

https://github.com/liquity/bold/blob/3f190ec5d63fa26a64fa5edc9404b92e9e053e03/contracts/src/Zappers/Modules/Exchanges/CurveExchange.sol#L57-L76

```solidity

    function swapFromBold(uint256 _boldAmount, uint256 _minCollAmount, address _zapper) external returns (uint256) {
        ICurvePool curvePoolCached = curvePool;
        IBoldToken boldTokenCached = boldToken;
        uint256 initialBoldBalance = boldTokenCached.balanceOf(address(this));
        boldTokenCached.transferFrom(_zapper, address(this), _boldAmount);
        boldTokenCached.approve(address(curvePoolCached), _boldAmount);

        // TODO: make this work
        //return curvePoolCached.exchange(BOLD_TOKEN_INDEX, COLL_TOKEN_INDEX, _boldAmount, _minCollAmount, false, _zapper);
        uint256 output = curvePoolCached.exchange(BOLD_TOKEN_INDEX, COLL_TOKEN_INDEX, _boldAmount, _minCollAmount);
        collToken.safeTransfer(_zapper, output);

        uint256 currentBoldBalance = boldTokenCached.balanceOf(address(this));
        if (currentBoldBalance > initialBoldBalance) {
            boldTokenCached.transfer(_zapper, currentBoldBalance - initialBoldBalance);
        }

        return output;
    }
```

As already flagged, these calls allow anybody to "create volume" for the zapper (or any approved addresses)
Through imbalancing the pool, these arbitrary calls can be used to cause a loss of value to the victim, at the gain of the caller

I highly recommend changing these functions to use `msg.sender` or to not use allowance at all (have the caller transfer then call the swap function)

**QA - Inconsistent usage of `transfer` vs `safeTransfer`**

I believe all collaterals are fine to use `transfer`, however the usage is inconsistent

I always recommend adding a comment anytime you use `transfer` to prevent wasted time by reviewers

**Gas - You don't need to cache `immutable` variables**

**Gas - You can deposit 1 unit of dust token to each Zap to make operations cheaper on average**

---
### Example 2

**Auto Label:** Insufficient input validation leads to unauthorized operations, fund loss, or state manipulation through unchecked external calls or missing value verification.  

**Original Text Preview:**

##### Description

The return value of an external call is not stored in a local or state variable.

* **Code Location**

```
    function _supportMarket(MToken mToken) external returns (uint) {
        if (msg.sender != admin) {
            return fail(Error.UNAUTHORIZED, FailureInfo.SUPPORT_MARKET_OWNER_CHECK);
        }

        if (markets[address(mToken)].isListed) {
            return fail(Error.MARKET_ALREADY_LISTED, FailureInfo.SUPPORT_MARKET_EXISTS);
        }

        mToken.isMToken(); // Sanity check to make sure its really a MToken

        Market storage newMarket = markets[address(mToken)];
        newMarket.isListed = true;
        newMarket.collateralFactorMantissa = 0;

        _addMarketInternal(address(mToken));

        emit MarketListed(mToken);

        return uint(Error.NO_ERROR);
    }

```

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:F/S:C (0.0)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:F/S:C)

##### Recommendation

It is recommended to use a `require` statement due to the return value of an external call is not stored in a local or state variable.

  

### Remediation Plan

**SOLVED**: The `Moonwell Finance team` solved the issue by using `require` statement.

---
### Example 3

**Auto Label:** Insufficient access control and bounds checking enable unauthorized asset withdrawals or permanent fund locking, leading to high-impact liquidity drain or user fund trapping.  

**Original Text Preview:**

## Context: Liquidation.sol#L80

## Description
Self liquidation with sub-accounts is allowed, which can enable "future unknown protocol attacks". The Euler v1 hack was enabled by two flaws in its system:

1. First, an issue with `donateToReserves` allowed the attacker to put themselves in a liquidatable state.
2. Second, the system did not prevent self liquidations.

By leveraging these two problems, the attacker was able to liquidate themselves for profit, draining the whole protocol.

In Euler v2, the developers decided to disallow self-liquidations to prevent similar attacks, but they failed to account for sub-accounts. As a result, sub-accounts are allowed to liquidate other sub-accounts controlled by the same owner. Although this by itself does not create any issues leading to loss of funds, it is a prerequisite of a major hack that happened in the past. Since Euler is putting a lot of effort into preventing "future unknown protocol attacks", as stated in the EVK whitepaper, it is advisable that this door is also closed to reduce the attack surface of the system.

Since this vulnerability does not explain a clear path leading to loss of funds, it is classified as a Low severity issue.

## Recommendation
Do not allow sub-accounts from the same owner to liquidate themselves.

---
