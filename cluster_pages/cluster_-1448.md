# Cluster -1448

**Rank:** #159  
**Count:** 84  

## Label
Inconsistent state validation and incomplete access controls enable attackers to manipulate token balances, bypass safeguards, or extract funds—leading to fund loss, supply overruns, or systemic risk.

## Cluster Information
- **Total Findings:** 84

## Examples

### Example 1

**Auto Label:** Inconsistent state validation and incomplete access controls enable attackers to manipulate token balances, bypass safeguards, or extract funds—leading to fund loss, supply overruns, or systemic risk.  

**Original Text Preview:**

The `cap` variable is intended to limit the total supply of `rsTAO` and `rsCOMAI`, and is checked in the `wrap()` function to ensure that the total minted does not exceed this cap:

```solidity
File: RivusTAO.sol
887:   function wrap(uint256 wtaoAmount) public nonReentrant checkPaused returns (uint256) {
...
...
894:     require(
895:       cap >= totalRsTAOMinted,
896:       "Deposit amount exceeds maximum"
897:     );
...
...
```

However, the current implementation allows the `cap` to be exceeded under certain conditions. Consider the following scenario:

1. The `cap` is set to `10` and `totalRsTAOMinted` is `9`.
2. A user performs a `wrap()` operation that would mint `30 rsTAO`, resulting in `totalRsTAOMinted` becoming `39`, thereby exceeding the `cap` of `10`.

This can happen because the `cap` check is performed before the new `rsTAO` is minted and added to the `totalRsTAOMinted`.

It is advisable to adjust the validation logic to include the amount of `rsTAO` to be minted in the cap check. This can be done by moving the cap check to after the calculation of `rsTAO` to be minted. This ensures the cap is not exceeded after new tokens are minted:

```diff
  function wrap(uint256 wtaoAmount) public nonReentrant checkPaused returns (uint256) {
    ...
    ...
--  require(
--    cap >= totalRsTAOMinted,
--    "Deposit amount exceeds maximum"
--  );
    ...
    ...
    uint256 rsTAOAmount = getRsTAObyWTAO(wrapAmountAfterFee);

    // Perform token transfers
    _mintRsTAO(msg.sender, rsTAOAmount);
++  require(
++    cap >= totalRsTAOMinted,
++    "Deposit amount exceeds maximum"
++  );
    ...
    ...
  }
```

---
### Example 2

**Auto Label:** Inconsistent state validation and incomplete access controls enable attackers to manipulate token balances, bypass safeguards, or extract funds—leading to fund loss, supply overruns, or systemic risk.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/720 

## Found by 
aman, future, rudhra1749

### Summary
The cross-chain borrow flow is as follow:
1. On Chain A (source chain), calculate the collateral value and send it.
2. On Chain B (destination chain), borrow the asset and register it to `crossChainCollateral`.
3. On Chain A, register the debt to `crossChainBorrow`.
During this process, the underlying token of the source chain and the lToken of the destination chain are used for `srcToken` and `dstlToken` of `crossChainCollateral` and `crossChainBorrow`. 

The cross-chain liquidation flow is as follow:
1. Initiation on Chain B.
2. Seizure on Chain A (the source chain).
3. Repayment on Chain B (the destination chain), or handle failure.
4. Matching(DestRepay) on Chain A.

### Root Cause
In Cross-Chain Liquidation Flow Step 3, during the identification of `crossChainCollaterals`, the `srcToken` field incorrectly uses the destination chain's borrowed token instead of the source chain's borrowed token.

### Internal pre-conditions
N/A

### External pre-conditions
N/A

### Attack Path
N/A

### PoC
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L804-L821
```solidity
    function _send(
        uint32 _dstEid,
        uint256 _amount,
        uint256 _borrowIndex,
        uint256 _collateral,
        address _sender,
        address _destlToken,
        address _liquidator,
        address _srcToken,
        ContractType ctype
    ) ...
```

We can confirm that `crossChainCollaterals.srcToken` represents the underlying token on the source chain, while `crossChainCollaterals.borrowedlToken` corresponds to the lToken on the destination chain.

Borrow Step 1, on Chain A:
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L151
```solidity
    function borrowCrossChain(uint256 _amount, address _borrowToken, uint32 _destEid) external payable {
        ...
        address destLToken = lendStorage.underlyingToDestlToken(_borrowToken, _destEid);
        ...
        _send(
            _destEid,
            _amount,
            0, // Initial borrowIndex, will be set on dest chain
            collateral,
            msg.sender,
            destLToken,
            address(0), // liquidator
151:        _borrowToken,
            ContractType.BorrowCrossChain
        );
    }
```
The _borrowToken is underlying token of chainA.

Borrow Step 2, on Chain B:
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L650
```solidity
    function _handleBorrowCrossChainRequest(LZPayload memory payload, uint32 srcEid) private {
        ...{
            lendStorage.addCrossChainCollateral(
                payload.sender,
                destUnderlying,
                LendStorage.Borrow({
                    srcEid: srcEid,
                    destEid: currentEid,
                    principle: payload.amount,
                    borrowIndex: currentBorrowIndex,
                    borrowedlToken: payload.destlToken,
650:                srcToken: payload.srcToken
                })
            );
        }...
    }
```
The payload.srcToken(srcToken of CrossChainCollateral) is underlying token of chainA.

Liquidation Step 1, on Chain B:
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L282
```solidity
    function _executeLiquidationCore(LendStorage.LiquidationParams memory params) private {
        // Calculate seize tokens
        address borrowedlToken = lendStorage.underlyingTolToken(params.borrowedAsset);
        ...
        _send(
            params.srcEid,
            seizeTokens,
            params.storedBorrowIndex,
            0,
            params.borrower,
            lendStorage.crossChainLTokenMap(params.lTokenToSeize, params.srcEid), // Convert to Chain A version before sending
            msg.sender,
282:        params.borrowedAsset,
            ContractType.CrossChainLiquidationExecute
        );
    }
```
Here, `params.borrowedAsset` is the underlying borrowed token on Chain B.

Liquidation Step 2, on Chain A:
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L363
```solidity
    function _handleLiquidationExecute(LZPayload memory payload, uint32 srcEid) private {
        ...
        _send(
            srcEid,
            payload.amount,
            0,
            0,
            payload.sender,
            payload.destlToken,
            payload.liquidator,
363:        payload.srcToken,
            ContractType.LiquidationSuccess
        );
    }
```
In this step, the payload.srcToken(above the `borrowedAsset`) is sent again to Chain B.

Liquidation Step 3, on Chain B:
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L453
```solidity
    function _handleLiquidationSuccess(LZPayload memory payload) private {
        // Find the borrow position on Chain B to get the correct srcEid
        address underlying = lendStorage.lTokenToUnderlying(payload.destlToken);

        // Find the specific collateral record
        (bool found, uint256 index) = lendStorage.findCrossChainCollateral(
            payload.sender,
            underlying,
            currentEid, // srcEid is current chain
            0, // We don't know destEid yet, but we can match on other fields
453:        payload.destlToken,
454:        payload.srcToken
        );

        require(found, "Borrow position not found");

        LendStorage.Borrow[] memory userCollaterals = lendStorage.getCrossChainCollaterals(payload.sender, underlying);
        uint32 srcEid = uint32(userCollaterals[index].srcEid);

        // Now that we know the borrow position and srcEid, we can repay the borrow using the escrowed tokens
        // repayCrossChainBorrowInternal will handle updating state and distributing rewards.
        repayCrossChainBorrowInternal(
            payload.sender, // The borrower
            payload.liquidator, // The liquidator (repayer)
            payload.amount, // Amount to repay
            payload.destlToken, // lToken representing the borrowed asset on this chain
            srcEid // The chain where the collateral (and borrow reference) is tracked
        );
    }
```
Here, `payload.srcToken`, which is the destination chain(chainB)'s borrowed token, is incorrectly used as the source chain's borrowed token.
Even if the asset is the same (e.g., USDC), the address on each chain could be different.
This could lead to failures in finding the borrow position and potentially cause the liquidation to revert.

Even if the seize operation is successful on Chain A, the repayment could fail on Chain B. 
This means that while the liquidatee's collateral is seized, the liquidatee's debt remains unchanged.
This is loss of liquidatee and the protocol could also suffer losses due to under-water positions.

### Impact
1. Loss of users' funds.
2. Protocol's financial loss.

### Mitigation

---
### Example 3

**Auto Label:** Inconsistent state validation and incomplete access controls enable attackers to manipulate token balances, bypass safeguards, or extract funds—leading to fund loss, supply overruns, or systemic risk.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Medium

## Description

The protocol checks **total stablecoin value** when opening `PUT` positions and processing stablecoin withdrawals. However, **payouts during position exercise require a specific stablecoin (`pos.buyToken`)** to be available. This discrepancy can cause `PUT` positions to become unexercisable even when the overall pool appears solvent.

Consider the following scenarios:
Assume:

- `OperationalTreasury.baseSetUp.token`: `USDC` (`buyToken`)
- Pool's stablecoin: `USDC` (listed & reward), `USDXL` (listed)
- Stable coin price is 1:1

#### Scenario 1: PUT position opens with 0 reserved for its payout token

Assume:

- `poolAmount[USDC]` = 0 USDC -> 0 USD
- `poolAmount[USDXL]` = 3500 USDXL -> 3500 USD
- `_getPoolValue(true, false)` = 0 + 3500 = 3500 USD

1. User opens `PUT` position, resulting in `lockedUSD` = 3500 USD
   - The check passes: `availableStable = 3500 ≥ lockedUSD:0 + pos.sizeUSD: 3500`
   - This allows to create the `PUT` position that will pay `USDC` if position is profit but there are no `USDC` in the pool.

```solidity
function _checkEnoughLiquidity(
    //...
) internal pure {
    if (pos.opType == OptionType.CALL) {
        --- SNIPPED ---
    } else if (pos.opType == OptionType.PUT) {
@>      if (lockedUSD + pos.sizeUSD > availableStable) {
            revert NotEnoughLiquidityPut(pos.sizeUSD, availableStable, lockedUSD);
        }
    }
}
```

2. User attempts to exercise their `PUT` position, but if at that time the `poolAmount[USDC]` still holds 0 `USDC`, the `_payout()` reverts.

```solidity
function _payout(Position memory pos, PositionClose memory close) internal returns (uint256 pnl) {
    --- SNIPPED ---
@>  strg.ledger.state.poolAmount[pos.buyToken] -= pnl;
    _doTransferOut(pos.buyToken, msg.sender, pnl);
}
```

#### Scenario 2: LP withdrawal drains payout token after the position opened

Assume:

- `poolAmount[USDC]` = 2000 USDC -> 2000 USD
- `poolAmount[USDXL]` = 3500 USDXL -> 3500 USD
- `_getPoolValue(true, false)` = 2000 + 3500 = 5500 USD

1. User opens `PUT` position, resulting in `lockedUSD` = 3500 USD
   - The check passes: `availableStable = 5500 ≥ lockedUSD:0 + pos.sizeUSD: 3500`
2. Liquidity provider removes their liquidity and requests token output as `USDC`
   - LP amount worth 2000 `USDC` equivalent
   - `_checkWithdrawlImpact()` checks: `newUSDValuation = 5500 - 2000 = 3500 USD`, which is `newUSDValuation: 3500 >= lockedUSD: 3500`
   - The process allows to withdraw 2000 `USDC` amount
   - `poolAmount[USDC]` = 2000 - 2000 = 0 `USDC`

```solidity
function _checkWithdrawlImpact(
    //...
) internal view {
    --- SNIPPED ---
    else if (strg.setUp.assets.stablecoins.contains(_token)) {
        uint8 decimals = IERC20Metadata(_token).decimals();
        uint256 withdrawUSD = (_amount * _assetPrice).toDecimals(decimals + 8, 18);
@>      uint256 newUSDValuation = _getPoolValue(true, false) - withdrawUSD;

@>      if (newUSDValuation < strg.ledger.state.lockedUSD) revert PoolErrors.InsufficientPoolAmount(_token);
    }
}
```

3. User attempts to exercise their `PUT` position, but `poolAmount[USDC]` = 0 `USDC`, so the `_payout()` reverts.

## Recommendation

Track per-buyToken (stablecoin) locked amounts and validate against individual balances in both `PUT` position opening and withdrawal impact checks.

---
