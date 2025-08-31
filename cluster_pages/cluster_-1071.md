# Cluster -1071

**Rank:** #344  
**Count:** 18  

## Label
**Inaccurate asset accounting due to unadjusted slashing events, leading to overstatement of total assets, distorted share prices, and irreversible fund misrepresentation.**

## Cluster Information
- **Total Findings:** 18

## Examples

### Example 1

**Auto Label:** **Incorrect asset accounting due to missing state updates and improper balance tracking, leading to fund loss, inflated reserves, and compromised protocol integrity.**  

**Original Text Preview:**

##### Description
The `FORCE_RELEASER_ROLE` can call `TreasuryIntermediateEscrow.forceRelease()` with a `_payoutAmount` of zero multiple times. This leads to a decrease in the `lockedBalances` mapping for a specific asset by an amount greater than the `depositedAmount` of a given `_escrowId`. As a result, `lockedBalances` will decrease faster than expected and may underflow when the last locked entry is released.

https://github.com/resolv-im/resolv-contracts/blob/91c52d9801c6a37f39566e2854722be1700dedc0/contracts/TreasuryIntermediateEscrow.sol#L111

##### Recommendation
We recommend adding a restriction that `_payoutAmount` is not equal to zero in `TreasuryIntermediateEscrow.forceRelease()`.

---
### Example 2

**Auto Label:** **Inaccurate asset accounting due to unadjusted slashing events, leading to overstatement of total assets, distorted share prices, and irreversible fund misrepresentation.**  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-08-saffron-finance-judging/issues/105 

## Found by 
0x73696d616f
### Summary

The parameters of each `LidoVault` are tuned such that fixed depositors get an upfront premium and variable depositors in return get all the yield produxed by the fixed depositors' deposits.

However, the protocol does not account for the fact that Lido may be DoSed for up to 36 days if it enters [bunker](https://blog.lido.fi/just-how-fast-are-ethereum-withdrawals-using-the-lido-protocol/) mode. Assuming the return is 4% a year, users are losing approximately `4 * 36 / 365 == 0.4 %`, which goes against the intended returns of the protocol.

Additionally, an attacker may forcefully trigger this by transferring only [up](https://github.com/sherlock-audit/2024-08-saffron-finance/blob/main/lido-fiv/contracts/LidoVault.sol#L712) to 100 wei of steth, which will make the protocol request an withdrawal and be on hold for 36 days.

The protocol should allow users to withdraw by swapping or similar, taking a much lower amount such as 0.12%, described [here](https://blog.lido.fi/just-how-fast-are-ethereum-withdrawals-using-the-lido-protocol/).

### Root Cause

In `LidoVault:712`, anyone may transfer just 100 wei of steth and DoS the protocol, so fixed, variable and the owner can not withdraw their funds for up to 36 days.

### Internal pre-conditions

None.

### External pre-conditions

Lido enters bunker mode, which is in scope as it happens when a mass slashing event happens, which is in scope
> The Lido Liquid Staking protocol can experience slashing incidents (such as this https://blog.lido.fi/post-mortem-launchnodes-slashing-incident/). These incidents will decrease income from deposits to the Lido Liquid Staking protocol and could decrease the stETH balance. The contract must be operational after it

### Attack Path

1. Vault has already requested all withdrawals, but they have not yet been claimed, so funds are in the protocol but it does not hold stEth anymore.
2. Attacker transfers 100 wei of steth, triggering the [request](https://github.com/sherlock-audit/2024-08-saffron-finance/blob/main/lido-fiv/contracts/LidoVault.sol#L722) of this 100 steth.
3. All funds are DoSed for 36 days.

### Impact

36 days DoS, which means the protocol can not get the expected interest rate calculated.

### PoC

Look at the function `LidoVault::vaultEndedWithdraw()` for confirmation.

### Mitigation

Firstly, an attacker should not be able to transfer 100 wei of steth and initiate a request because of this. The threshold should be computed based on an estimated earnings left to withdraw for variable depositors and fixed depositors that have not claimed, not just 100.

Secondly, it would be best if there was an alternative way to withdraw in case requests are taking too much.

---
### Example 3

**Auto Label:** **Inaccurate asset accounting due to unadjusted slashing events, leading to overstatement of total assets, distorted share prices, and irreversible fund misrepresentation.**  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-08-saffron-finance-judging/issues/92 

The protocol has acknowledged this issue.

## Found by 
0x73696d616f
### Summary

In [LidoVault::withdraw()](https://github.com/sherlock-audit/2024-08-saffron-finance/blob/main/lido-fiv/contracts/LidoVault.sol#L423), when the vault has started but not ended, it limits the value to withdraw if a slashing event occured and withdraws `lidoStETHBalance.mulDiv(fixedBearerToken[msg.sender], fixedLidoSharesTotalSupply());`. However, it also decreases `fixedSidestETHOnStartCapacity` by this same amount, which means that next users that withdraw will get more than their initial deposit in case the Lido ratio comes back up (likely during the vault's duration).

It's clear from the code users should get exactly their initial amount of funds or less, never more, as the [comment](https://github.com/sherlock-audit/2024-08-saffron-finance/blob/main/lido-fiv/contracts/LidoVault.sol#L486) indicates:
> since the vault has started only withdraw their initial deposit equivalent in stETH  at the start of the vault- unless we are in a loss

### Root Cause

In `LidoVault.sol:498`, `fixedSidestETHOnStartCapacity` is decreased by a lower amount than it should.

### Internal pre-conditions

None.

### External pre-conditions

Lido slash, which is in scope as per the readme.
> The Lido Liquid Staking protocol can experience slashing incidents (such as this https://blog.lido.fi/post-mortem-launchnodes-slashing-incident/). These incidents will decrease income from deposits to the Lido Liquid Staking protocol and could decrease the stETH balance. The contract must be operational after it

### Attack Path

1. Lido slashes, decreasing the steth ETH / share ratio.
2. User withdraws, taking a loss and decreasing `fixedSidestETHOnStartCapacity` with the lossy amount.
3. Next user withdrawing will withdraw more because `fixedSidestETHOnStartCapacity` will be bigger than it should.

### Impact

Fixed deposit users benefit from the slashing event at the expense of variable users who will take the loss.

### PoC

Assume that there 100 ETH and 100 shares.
A slashing event occurs and drops the ETH to 90 and shares remain 100.
There are 2 fixed depositors, with 50% of the deposits each.
User A withdraws, and should take `100 ETH * 50 / 100 == 50 ETH`, but takes `90 ETH * 50 / 100 == 45 ETH` instead due to the loss.
`fixedSidestETHOnStartCapacity` is decreased by `45 ETH`, the withdrawn amount, so it becomes `55 ETH`.
Now, when LIDO recovers from the slashing, the contract will hold more steth than `fixedSidestETHOnStartCapacity`, more specifically the remaining 45 ETH in the contract that were not withdrawn yet are worth 50 ETH now. So user B gets
`fixedSidestETHOnStartCapacity * 50 / 50 == 55`. 

As the fixed deposit user initially deposited 50, but claimed 55 now, it is getting much more than it should at the expense of the variable users who will take the loss.

### Mitigation

The `fixedSidestETHOnStartCapacity` should be always reduced by `fixedETHDeposits.mulDiv(fixedBearerToken[msg.sender], fixedLidoSharesTotalSupply());`, such that users get their equivalent ETH from their initial deposit back and the variable users don't take losses.

---
