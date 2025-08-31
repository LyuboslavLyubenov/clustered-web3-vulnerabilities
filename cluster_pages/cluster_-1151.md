# Cluster -1151

**Rank:** #122  
**Count:** 128  

## Label
Inadequate state validation and access control lead to incorrect balance tracking and denial-of-service, enabling attackers to withdraw inflated funds through unchecked token duplication and unauthorized state reads.

## Cluster Information
- **Total Findings:** 128

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

**Auto Label:** Missing or incorrect balance validation leads to state inconsistency, enabling attackers to manipulate token balances, trigger unintended behavior, or exploit race conditions and arithmetic flaws.  

**Original Text Preview:**

In the current implementation, the protocol's token removal functionality can be permanently blocked by a malicious actor through a minimal token deposit. This vulnerability exists due to the strict validation in PoolAdminFacet.sol:

```solidity
    function removeToken(address _token) external onlyRole(AccessControlStorage.DEFAULT_ADMIN_ROLE) {
        PoolStorage.Layout storage strg = PoolStorage.layout();
        if (strg.ledger.state.poolAmount[_token] != 0) revert PoolErrors.PoolNotEmpty(_token);
        strg.setUp.assets.allAssets.remove(_token);
        strg.setUp.assets.stablecoins.remove(_token);

        if (strg.ledger.state.rewardsByToken[_token].totalRewards == 0) strg.setUp.assets.rewardTokens.remove(_token);
    }
```

Any user can deposit a minimal amount (as small as 1 `wei`) of a token into the pool. Once a token has any non-zero balance, the admin cannot remove it from the protocol. This creates a permanent denial-of-service (DOS) vector for token removal functionality.

**This issue is more severe if one of the stable coins depends. Being Unable to remove the coin from the `strg.setUp.assets.stablecoins` could cause an incorrect `getPoolValue` calculation.**

Recommendations:

Implement a more robust token removal system that allows administrators to remove tokens even with minimal balance.

---
