# Cluster -1433

**Rank:** #297  
**Count:** 25  

## Label
Insufficient slippage protection due to flawed price assumptions and premature caps, enabling attackers to exploit price volatility and supply dynamics, resulting in users receiving fewer tokens than expected.

## Cluster Information
- **Total Findings:** 25

## Examples

### Example 1

**Auto Label:** Failure to enforce slippage limits due to unreliable or mutable price data and inadequate real-time validation, leading to unauthorized trades, financial loss, or fund theft.  

**Original Text Preview:**

In function buy(), we should add two safety checks on slippagePercent:

1. We should ensure it is not greater than 10000.
2. We should ensure that the BUYER_ROLE cannot misuse the parameter to set slippage to the absolute maximum i.e. 10000. For this, consider allowing the DEFAULT_ADMIN_ROLE to set an allowable limit that the BUYER_ROLE cannot exceed.

```solidity
function buy(uint256 amountIn, uint256 slippagePercent, bool useUSDC) external onlyRole(BUYER_ROLE) {
```

---
### Example 2

**Auto Label:** Flawed slippage protection mechanisms due to incorrect validation, arithmetic errors, or boundary handling, leading to unauthorized fund exposure or loss under market volatility.  

**Original Text Preview:**

**Description:** When the BENQI admin sets the `slippage` state by calling [`StakingContract::setSlippage`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L313-328), it is validated that this new value is between the `minSlippage` and `maxSlippage` thresholds:

```solidity
require(
    _slippage >= minSlippage && _slippage <= maxSlippage,
    "Slippage must be between min and max"
);
```

The admin can also change both `minSlippage` and `maxSlippage` via [`StakingContract::setMinSlippage`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L330-345) and [`StakingContract::setMaxSlippage`](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/blob/b63336201f50f9a67451bf5c7b32ddcc4a847ce2/contracts/staking.sol#L347-360); however, neither of these functions has any validation that the `slippage` state variable remains within the boundaries.

**Impact:** Incorrect usage of either `StakingContract::setMaxSlippage` or `StakingContract::setMinSlippage` can result in the `slippage` state variable being outside the range.

**Proof of Concept:** The following test can be added to `stakingContract.test.js` under `describe("setSlippage")`:

```javascript
it("can have slippage outside of max and min", async function () {
    // slippage is set to 4
    await stakingContract.connect(benqiAdmin).setSlippage(4);
    // max slippage is updated below it
    await stakingContract.connect(benqiAdmin).setMaxSlippage(3);

    const slippage = await stakingContract.slippage();
    const maxSlippage = await stakingContract.maxSlippage();
    expect(slippage).to.be.greaterThan(maxSlippage);
});
```

**Recommended Mitigation:** Consider validating that the `slippage` state variable is within the boundaries set using `setMin/MaxSlippage()`.

**BENQI:** Fixed in commits [96e1b96](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/commit/96e1b9610970259cffdf44fd7cf1af527016b0ce) and [dbb13c5](https://code.zeeve.net/zeeve-endeavors/benqi_smartcontract/-/commit/dbb13c55047e4ce52e39f833196fc78ed5c0cf8a).

**Cyfrin:** Verified.

---
### Example 3

**Auto Label:** Insufficient slippage protection due to flawed price assumptions and premature caps, enabling attackers to exploit price volatility and supply dynamics, resulting in users receiving fewer tokens than expected.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Low

## Description

When users buy tokens using the `buyWithReferral` function, they specify the token amount to buy and send the corresponding amount of ETH. There isn’t slippage protection due to the assumption that the maximum price users will pay is `tokenAmountToBuy/msg.value`, allowing users to control the price they pay.

However, users can still end up paying a higher price than expected (slippage) because:

- The number of tokens available for purchase might be less than the `tokenAmountToBuy` if the remaining token supply is less than the `tokenAmountToBuy`.
- As the available token supply decreases, the price of the token increases.

In a scenario where a user intends to buy 10e18 tokens with 1 ETH (price is 0.1 ETH per 1e18 tokens), but another user front-runs and buys 9e18 tokens before them, the initial user will only receive 1e18 tokens and end up paying more than 0.1 ETH, which is more than the intended price of 0.1 ETH per 1e18 tokens.

```solidity
    function buyWithReferral(address erc20Address, uint256 tokenAmountToBuy, address referral)
        public
        payable
        nonReentrant
        validToken(erc20Address)
    {
        ...
        uint256 tokenAmountLeft = getTokenAmountLeft(erc20Address);
        if (tokenAmountToBuy >= tokenAmountLeft) { // @audit amount to buy can be less than specified
            tokenAmountToBuy = tokenAmountLeft;
            // pause the curve when all available tokens have been bought
            curves[erc20Address].isPaused = true;
            emit BondingCurveCompleted(erc20Address);
        }
        ...
    }
```

## Recommendations

Allow users to specify the minimum amount of tokens they will receive to avoid slippage.

---
