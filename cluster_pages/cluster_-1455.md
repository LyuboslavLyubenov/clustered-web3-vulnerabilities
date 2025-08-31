# Cluster -1455

**Rank:** #417  
**Count:** 11  

## Label
Attackers manipulate asset balances via donations to exploit rounding or initialization flaws, artificially inflating share prices or reducing share supply, leading to unfair advantages, asset loss, or price manipulation in ERC-4626-based vaults.

## Cluster Information
- **Total Findings:** 11

## Examples

### Example 1

**Auto Label:** Attackers manipulate asset balances via donations to exploit rounding or initialization flaws, artificially inflating share prices or reducing share supply, leading to unfair advantages, asset loss, or price manipulation in ERC-4626-based vaults.  

**Original Text Preview:**

Code uses `convertToAssets()` and `convertToShares()` to calculate token conversion amounts. The issue is that ERC4626 vaults share price can be manipulated by donation attacks and as result, attackers can perform price manipulation attacks. Performing the attack is costly and the attack would be profitable only in special circumstances like using ERC4626 as collateral.

---
### Example 2

**Auto Label:** Attackers manipulate asset balances via donations to exploit rounding or initialization flaws, artificially inflating share prices or reducing share supply, leading to unfair advantages, asset loss, or price manipulation in ERC-4626-based vaults.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-04-burve-judging/issues/387 

## Found by 
TessKimy, m3dython, x0lohaclohell

### Summary

The absence of protective mechanisms in the [NoopVault](https://github.com/sherlock-audit/2025-04-burve/blob/main/Burve/src/integrations/pseudo4626/noopVault.sol#L6) as a vertex vault allows an attacker to exploit the ERC4626 standard's vulnerability to donation attacks. By front-running the first legitimate deposit and donating assets directly(without interacting with the valueFacet) to the vault without receiving shares, the attacker can manipulate the share-to-asset ratio. This manipulation enables the attacker to withdraw a disproportionate amount of assets, effectively draining each vertex in a  Closure.​


### Root Cause

There is no mechanism to prevent donation attacks, such as initializing the vault with a non-zero share supply or implementing virtual assets/shares. This omission allows an attacker to manipulate the vault's share-to-asset ratio by donating assets without receiving corresponding shares.
```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity ^0.8.27;
import "openzeppelin-contracts/token/ERC20/extensions/ERC4626.sol";
import "openzeppelin-contracts/token/ERC20/ERC20.sol";

contract NoopVault is ERC4626 {
    constructor(
        ERC20 asset,
        string memory name,
        string memory symbol
    ) ERC20(name, symbol) ERC4626(asset) {}
}

```

### Internal Pre-conditions

1. Admin adds a new NoopVault as a vertex in the Burve protocol.
2. The vault has zero total supply and total assets upon initialization.
3. No initial deposit is made to the vault to set a baseline share-to-asset ratio.

### External Pre-conditions

1. Berachain, being an L1 blockchain, has a transparent mempool where pending transactions can be observed.
2. An attacker monitors the mempool for transactions adding new vertices (vaults) to the Burve protocol.

### Attack Path

1. Attacker observes a transaction in the mempool adding a new Vault as a vertex.
2. Before any legitimate user adds value through the valueFacet, the attacker front-runs by depositing a minimal amount (e.g., 1 wei) directly into the vault, receiving all the initial shares.
3. The attacker then donates a significant amount of assets directly to the vault without minting new shares, inflating the total assets.
4. A legitimate user deposits assets into the vault, expecting to receive shares proportional to their deposit. However, due to the inflated asset pool and the attacker's control of all shares, the user receives fewer shares than expected.
5. The attacker redeems their shares, withdrawing a disproportionate amount of assets from the vault, effectively draining the Closure.​

### Impact

Users suffer significant losses as their deposits yield fewer shares than expected, leading to a loss in asset value and the attacker gains assets equivalent to the legitimate users deposits without providing corresponding value. This attack can be cordinated to target all vertices in a closure and drain the whole closure.

### PoC

_No response_

### Mitigation

Implement Virtual Assets/Shares: Incorporate virtual assets and shares in the vault's accounting to simulate an initial balance, mitigating the impact of donation attacks.

## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/itos-finance/Burve/pull/79

---
### Example 3

**Auto Label:** Attackers manipulate asset balances via donations to exploit rounding or initialization flaws, artificially inflating share prices or reducing share supply, leading to unfair advantages, asset loss, or price manipulation in ERC-4626-based vaults.  

**Original Text Preview:**

##### Description

The ratio computation mechanism within the vault is susceptible to manipulation through a "donation attack." This attack vector is realized when a significant amount of tokens, equal to or exceeding `1e18`, is donated to the vault before any other transaction. Subsequent to this donation, the share price, or ratio, becomes permanently fixed at `1`, irrespective of the actual amount of deposited assets or the total supply of shares within the vault. The standard calculation method using `Convert.multiplyAndDivideCeil` rounds up and solidifies the ratio at `1`, regardless of subsequent transactions, thus effectively breaking the dynamic share pricing mechanism intended by the ERC4626 standard.

The persistent ratio of `1` distorts the share-to-asset conversion process, causing deposits less than `1e18` to round down to zero shares minted, leading to a total loss of the deposited assets for the user. This distortion prevents the vault from correctly accounting for new deposits, disrupting the intended proportional relationship between shares and underlying assets. It also introduces a vulnerability where the vault can no longer accurately represent the economic value of deposits and withdrawals, thereby undermining its functionality and user trust.

##### Proof of Concept

The following test has been added to `inceptionVaultV2.js` :

```
it("Donation Attack", async function () {
        console.log("----------- Donation Attack -----------");
        await iVault.connect(staker).deposit(100n, staker.address);
        console.log(`iToken totalSupply  : ${await iToken.totalSupply()}`);
        console.log(`total Assets 1      : ${await iVault.totalAssets()}`);
        console.log(`getTotalDeposited 1 : ${await iVault.getTotalDeposited()}`);
        console.log(`Ratio      1        : ${await iVault.ratio()}`);
        
        console.log("--- Attacker Donation 99stEth ---");
        await asset.connect(staker).transfer(iVault.getAddress(),99n * e18);
        console.log(`getTotalDeposited   : ${await iVault.getTotalDeposited() }`);
        console.log(`Ratio      2        : ${await iVault.ratio()}`);

        console.log("--- Victims Deposit ---");
        let deposited2 = 1n * e18; 
        let upTo = 8n;
        for (let i = 0; i < upTo; i++) {
          await iVault.connect(staker2).deposit(deposited2, staker2.address);
          await iVault.connect(staker3).deposit(deposited2, staker3.address);
        }
        console.log(`Ratio      3        : ${await iVault.ratio()}`);
        console.log(`total Assets 3      : ${await iVault.totalAssets()}`);
        console.log(`Staker 1 Shares     : ${await iToken.balanceOf(staker)}`);
        console.log(`Staker 2 Deposit    : ${deposited2 * upTo / e18} eth`);
        console.log(`Staker 2 Shares     : ${await iToken.balanceOf(staker2)}`);
        console.log(`Staker 3 Deposit    : ${deposited2 * upTo / e18} eth`);
        console.log(`Staker 3 Shares     : ${await iToken.balanceOf(staker3)}`);
        
        console.log("--- Attacker Withdraw ---");
        const tx3 = await iVault.connect(staker).withdraw(99n, staker.address);
        const receipt3 = await tx3.wait();
        const events3 = receipt3.logs?.filter((e) => e.eventName === "Withdraw");
        let amount = events3[0].args["amount"] ;
        console.log(`Amount redeemed to attacker : ${amount/e18}`);
      });

```

  

Result :

![resultVault.png](https://halbornmainframe.com/proxy/audits/images/662bd237ef90df57b51391e2)

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:N/A:N/D:C/Y:N/R:N/S:C (10.0)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:N/A:N/D:C/Y:N/R:N/S:C)

##### Recommendation

To mitigate this vulnerability, it is advised to:

* **Pre-Mint Shares**: Establish the vault with an initial deposit of shares that reflect a non-trivial amount of the underlying asset, preventing share price manipulation from being economically feasible.
* **Guard Against Zero Share Minting**: Implement safeguards within the deposit logic to reject any deposit transaction that would result in zero shares being minted. This could involve verifying the expected number of shares before completing the deposit transaction.

Remediation Plan
----------------

**SOLVED :** The **Tagus Labs team** implemented a check to ensure depositors get some shares when depositing LST tokens + all vaults are already deployed and already have a TVL in mainnet.

##### Remediation Hash

<https://github.com/inceptionlrt/smart-contracts/commit/0b33cca999f2ed1e29125b61376386507da4404c>

##### References

[inceptionlrt/smart-contracts/contracts/Inception/vaults/InceptionVault.sol#L329](https://github.com/inceptionlrt/smart-contracts/blob/inception-v2/contracts/Inception/vaults/InceptionVault.sol#L329)

---
