# Cluster -1229

**Rank:** #247  
**Count:** 38  

## Label
Front-running attacks exploit transaction ordering and inadequate fee controls, allowing malicious actors to profit from or disrupt legitimate claims by manipulating gas pricing and reward mechanisms.

## Cluster Information
- **Total Findings:** 38

## Examples

### Example 1

**Auto Label:** Front-running and race conditions enable attackers to manipulate transaction ordering or dynamic values, leading to denial-of-service, price manipulation, or unauthorized fund withdrawals.  

**Original Text Preview:**

**Description:** The [unbond](https://github.com/stakedotlink/contracts/blob/1d3aa1ed6c2fb7920b3dc3d87ece5d1645e1a628/contracts/polygonStaking/PolygonStrategy.sol#L213-L263) function in `PolygonStrategy` is vulnerable to DoS when small reward amounts are used to cover the remaining amount to withdraw.

When [PolygonFundFlowController.unbondVaults](https://github.com/stakedotlink/contracts/blob/1d3aa1ed6c2fb7920b3dc3d87ece5d1645e1a628/contracts/polygonStaking/PolygonFundFlowController.sol#L121) is called it passes the amount to withdraw/unbond to `PolygonStrategy.unbond`:
```solidity
    function unbondVaults() external {
       ...
        uint256 toWithdraw = queuedWithdrawals - (queuedDeposits + validatorRemovalDeposits);
@>        strategy.unbond(toWithdraw);
        timeOfLastUnbond = uint64(block.timestamp);
    }
```

The `unbond` function loops through validators, using their rewards to cover the withdrawal amount. If rewards are insufficient, it unbonds from the validator's principal deposits.
```solidity
function unbond(uint256 _toUnbond) external onlyFundFlowController {
   ...
    if (rewards >= toUnbondRemaining) {
        // @audit withdrawRewards could be below the threshold from ValidatorShares
@>        vault.withdrawRewards();
        toUnbondRemaining = 0;
    } else {
        toUnbondRemaining -= rewards;
        uint256 vaultToUnbond = principalDeposits >= toUnbondRemaining
            ? toUnbondRemaining
            : principalDeposits;
@>        vault.unbond(vaultToUnbond);
        toUnbondRemaining -= vaultToUnbond;
        ++numVaultsUnbonded;
    }
   ...
}
```

A common, though infrequent, scenario occurs when iteration leaves `toUnbondRemaining` with a small amount for the next validator. This amount may be coverable by the validator's rewards, but if those rewards are below the threshold in the [ValidatorShares](https://etherscan.io/address/0x7e94d6cAbb20114b22a088d828772645f68CC67B#code#F1#L225) contract, the transaction will revert, causing a DoS in the `unbond` process.

**Example:**
1. Strategy has 3 validators.
2. Validator 3 has accumulated 0.9 tokens in rewards
3. The system needs to unbond 30.5 tokens in total
4. Validators 1 and 2 cover 30 tokens with their unbonds
5. Validator 3 is expected to cover the remaining 0.5 tokens with its 0.9 rewards(`withdrawRewards` is triggered)
6. The transaction reverts because 0.9 tokens is less than minAmount from ValidatorShares

The current logic also allows a malicious user to DoS the `unbond`  transaction due to the way rewards are calculated in the `unbound` function:

```solidity
uint256 deposits = vault.getTotalDeposits();
uint256 principalDeposits = vault.getPrincipalDeposits();
uint256 rewards = deposits - principalDeposits;
```
The `getTotalDeposits` accounts for the current token balance in the `PolygonVault` contract.  If the `toUnbondRemaining` and validator's rewards are less than vault's minimum rewards, an attacker can front-run the transaction and cause `unbond` to revert given the current scenario:

1. Attacker sent dust amount to the `PolygonVault`. Enough to cover `toUnbondRemaining` but < 1e18 (minimum rewards).
2. `rewards = deposits - principalDeposits` >= `toUnbondRemaining`.
3. Vault will call `withdrawRewards` but current claim amount to be withdrawn < `1e18`(trigger for the revert in [ValidatorShares](https://etherscan.io/address/0x7e94d6cAbb20114b22a088d828772645f68CC67B#code#F1#L225))
4. Transaction reverts.

**Impact:** DoS in the unbonding process when small rewards (< 1e18) are needed to complete withdrawals, blocking the protocol's withdrawal flow.

**Proof of Concept:**
1. First adjust the `PolygonValidatorShareMock` to reflect the same behavior as `ValidatorShares` by adding a minimum amount requirement when withdrawing rewards.
```diff
// PolygonValidatorShareMock
   function withdrawRewardsPOL() external {
        uint256 rewards = liquidRewards[msg.sender];
        if (rewards == 0) revert NoRewards();
+        require(rewards >= 1e18, "Too small rewards amount");

        delete liquidRewards[msg.sender];
        stakeManager.withdraw(msg.sender, rewards);
    }
```

2. Paste the following test in `polygon-fund-flow-controller.test.ts` and `run npx hardhat test test/polygonStaking/polygon-fund-flow-controller.test.ts`:
```solidity
describe.only('DoS', async () => {
    it('will revert when withdrawRewards < 1e18', async () => {
      const { token, strategy, vaults, fundFlowController, withdrawalPool, validatorShare, validatorShare2, validatorShare3 } =
      await loadFixture(deployFixture)
      console.log("will print some stuff")

      // validator funds: [10, 20, 30]
      await withdrawalPool.setTotalQueuedWithdrawals(toEther(970.5));
      assert.equal(await fundFlowController.shouldUnbondVaults(), true);

      // 1. Pre-condition: validator acumulated dust rewards since last unbonding.
      await validatorShare3.addReward(vaults[2].target, toEther(0.9));

      // expect that vault 2 has rewards
      assert.equal(await vaults[2].getRewards(), toEther(0.9));

      // 2. Unbond:
      // Validator A will cover 10 with unbond
      // Validator B will cover 20 with unbond
      // Validator C will cover the remaining 0.5 with his 0.9 rewards.
      await expect(fundFlowController.unbondVaults()).to.be.revertedWith("Too small rewards amount");
    })
  })
```
Output:
```javascript
PolygonFundFlowController
    DoS
      ✔ will revert when withdrawRewards < 1e18 (800ms)

  1 passing (800ms)
```

**Recommended Mitigation:** In the `PolygonStrategy.unbond` check if the **actual** rewards that can be withdrawn from the `ValidatorShares` is greater than the min amount for claim(1e18) before calling `withdrawRewards`.
```diff
-if (rewards >= toUnbondRemaining) {
-    vault.withdrawRewards();
-    toUnbondRemaining = 0;
+if (rewards >= toUnbondRemaining && vault.getRewards() >= vault.minAmount()) {
+    vault.withdrawRewards();
+    toUnbondRemaining = 0;
} else {
+    if (toUnbondRemaining > rewards) {
+        toUnbondRemaining -= rewards;
+    }
-    toUnbondRemaining -= rewards;
     uint256 vaultToUnbond = principalDeposits >= toUnbondRemaining
         ? toUnbondRemaining
         : principalDeposits;
```


**Stake.Link:** Resolved in [PR 151](https://github.com/stakedotlink/contracts/pull/151/commits/fec5777db57e3649b27f8cff7ab2fd34b3fb9857)

**Cyfrin:** Resolved.

---
### Example 2

**Auto Label:** Front-running attacks exploiting timing gaps and insufficient validation, enabling attackers to manipulate rates, bypass slippage, or extract fees before legitimate users act.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Low

## Description

The UsdPlus Contract contains the `rebaseAdd`, `rebaseMul`, and `rebaseSub` functions, which allow an address with the `OPERATOR_ROLE` to perform a rebase. These functions adjust the value of `$._balancePerShare`, either increasing or decreasing it depending on the action executed.

```solidity
    function rebaseAdd(uint128 value) external onlyRole(OPERATOR_ROLE) {
        uint256 _supply = totalSupply();
        uint128 _balancePerShare = uint128(FixedPointMathLib.fullMulDiv(balancePerShare(), _supply + value, _supply));
        UsdPlusStorage storage $ = _getUsdPlusStorage();
        $._balancePerShare = _balancePerShare;
        emit BalancePerShareSet(_balancePerShare);
    }
```

This functionality can be exploited in the following ways:

Users can front-run a positive rebase to acquire tokens at a lower rate. Once the rebase is executed, they can submit a withdrawal request, locking in the higher rate. Although the redemption process can take up to three days, the `request` function locks the rate in the user’s request ticket, allowing them to secure profits.

Users can also front-run a negative rebase by submitting a withdrawal request before the rebase is executed, locking in the rate prior to the update. This allows them to avoid losses associated with the negative rebase, effectively bypassing the intended impact of the rebase adjustment.

## Recommendations

A small withdrawal fee could be implemented to make such exploit attempts less desirable. Additionally, a delay could also be introduced between deposit and withdrawal requests.

---
### Example 3

**Auto Label:** Front-running attacks exploit transaction ordering and inadequate fee controls, allowing malicious actors to profit from or disrupt legitimate claims by manipulating gas pricing and reward mechanisms.  

**Original Text Preview:**

## Context
(No context files were provided by the reviewer)

## Description
Any entity operating a mech can create simple requests and set the mech they are operating as the priority mech, allowing it to deliver right away and thus increase its `mapMechKarma`. On the other hand, `mapRequesterMechKarma` for a specific requester would be harder to manipulate since if the mech does not deliver quality data, the requester would stop creating requests that would have that specific mech as the priority mech (linked by the service id). Although this can also be manipulated if one considers the frontrunning issues related to `deliverMarketplaceWithSignatures`.

## Recommendation
A monitoring and analysis should be set up to shine a light on the above behaviors. It would also be best to document the above.

## Valory
There are several ways to game karma, and we acknowledge that we need to include this into our documentation.

## Cantina Managed
Acknowledged by Valory team.

---
