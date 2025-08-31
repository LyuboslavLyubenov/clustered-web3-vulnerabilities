# Cluster -1258

**Rank:** #161  
**Count:** 83  

## Label
Front-running exploits via mempool ordering enable attackers to profit from transaction timing gaps, manipulating prices, rewards, or state changes before legitimate users, leading to asymmetric gains and reward misallocation.

## Cluster Information
- **Total Findings:** 83

## Examples

### Example 1

**Auto Label:** Front-running exploits via mempool ordering enable attackers to profit from transaction timing gaps, manipulating prices, rewards, or state changes before legitimate users, leading to asymmetric gains and reward misallocation.  

**Original Text Preview:**

First see:

```solidity
function deposit(uint256 _usfAmount, address _receiver) public returns (uint256 wstUSFAmount) {
    wstUSFAmount = previewDeposit(_usfAmount);
    _deposit(msg.sender, _receiver, _usfAmount, wstUSFAmount);
    return wstUSFAmount;
}
```

```solidity
function withdraw(uint256 _usfAmount, address _receiver, address _owner) public returns (uint256 wstUSFAmount) {
    uint256 maxusfAmount = maxWithdraw(_owner);
    if (_usfAmount > maxusfAmount) revert ExceededMaxWithdraw(_owner, _usfAmount, maxusfAmount);
    wstUSFAmount = previewWithdraw(_usfAmount);
    _withdraw(msg.sender, _receiver, _owner, _usfAmount, wstUSFAmount);
    return wstUSFAmount;
}
```

As seen, yhe WstUSF contract allows users to deposit and withdraw within the same block without any timelock mechanism. This atomic interaction capability creates an attack vector where sophisticated users can frontrun reward distributions fromt the `RewardDistributor`.

Thats because when rewards are about to be distributed to the underlying StUSF contract (which WstUSF wraps), a malicious actor can:

1. Monitor the mempool for incoming reward distribution transactions
2. Frontrun these transactions with a large deposit to WstUSF
3. Receive a proportionally large share of the distributed rewards
4. Immediately withdraw their position in the same block
5. Extract value without maintaining a long-term position

The attack is particularly effective because of the ERC4626-style vault architecture that calculates share distribution based on point-in-time snapshots, though impact here is medium since not too much value can be leaked via this path, but this still reduces rewards for genuine long-term stakers and creates an unfair distribution mechanism that favors MEV-capable actors.

**Recommendation**

Consider implementing a timelock mechanism that enforces a minimum holding period between deposit and withdrawal operations.

---
### Example 2

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
### Example 3

**Auto Label:** Front-running attacks exploit transaction ordering in public mempools, allowing attackers to bypass restrictions or penalties by executing faster, state-dependent operations before intended enforcement mechanisms take effect.  

**Original Text Preview:**

**Description:** One unconventional application of regular transfers or cross-chain transfers via CCIP / LayerZero bridging is to evade the blocklist:
* user sees operator call to `MToken::addToBlockedList` in mempool which would block their address
* user front-runs this transaction by a normal transfer or a CCIP / LayerZero cross-chain transfer to bridge their tokens to a new `receiver` address on another chain
* if the operator attempts to call `MToken::addToBlockedList` on the other chain for the new `receiver` address, the user can bridge back to another new address again

To prevent this the operator can:
* pause bridging (pausing has been implemented for LayerZero but not CCIP) prior to calling `MToken::addToBlockedList`
* use a service such as [flashbots](https://www.flashbots.net/) when calling `MToken::addToBlockedList` so the transaction is not exposed in a public mempool

**Matrixdock:** Acknowledged.

---
