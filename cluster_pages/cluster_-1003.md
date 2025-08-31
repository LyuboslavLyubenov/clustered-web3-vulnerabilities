# Cluster -1003

**Rank:** #321  
**Count:** 21  

## Label
Failure to properly validate or update ownership and state during stake delegation and migration leads to unclaimed rewards, double-accounting, and inconsistent stake balances.

## Cluster Information
- **Total Findings:** 21

## Examples

### Example 1

**Auto Label:** Failure to properly validate or update ownership and state during stake delegation and migration leads to unclaimed rewards, double-accounting, and inconsistent stake balances.  

**Original Text Preview:**

##### Description
This issue has been identified within the `_delegateStake` function of the `HookTargetStakeDelegator` contract.
Currently, if the account owner is still unknown to the `checkVaultStatus()` call, `delegateStake()` [is executed](https://github.com/euler-xyz/evk-periphery/blob/995449e6e960b515869f00235fdcd417ad8e08fc/src/HookTarget/HookTargetStakeDelegator.sol#L278) in favor of the `account` itself. Any rewards earned by this account [can't be requested](https://github.com/berachain/contracts/blob/2a8de427bcebecc35c1e22679fd34a521473f15e/src/pol/rewards/RewardVault.sol#L326-L336) in the future. As a result, the portion of rewards earned between the delegation and the withdrawal from that account gets stuck on the `RewardVault` contract.
<br/>
##### Recommendation
We recommend avoiding minting and delegating tokens if the account owner is not yet known at the time of the `checkVaultStatus()` call. They can be accounted for using the one-time migration method proposed in the M-2 recommendation.

> **Client's Commentary:**
> We acknowledge the side effect described in the audit and have considered it during the design of the smart contract. Unfortunately, the recommended solution is not feasible. Implementing the recommendation would result in smart contracts potentially being unable to interact with Euler vaults. Typically, smart contracts do not interact with Euler vaults via the EVC (registering the owner) but can perform deposits and withdrawals directly on the vaults. Consequently, for smart contracts, it is likely that the true EVC owner is never known, but it can be safely assumed to be the account on which the given operation is performed (and to which `ERC20ShareRepresentation` tokens are delegate staked). Implementing the recommendation would disrupt smart contract integrations with Euler. The risk of rewards being potentially lost in case they are earned by the real non-owner account is known and accepted.

---

---
### Example 2

**Auto Label:** Failure to properly validate or update ownership and state during stake delegation and migration leads to unclaimed rewards, double-accounting, and inconsistent stake balances.  

**Original Text Preview:**

##### Description
This issue has been identified within the `checkVaultStatus()` function of the `HookTargetStakeDelegator` contract.
Stake migration [is only triggered](https://github.com/euler-xyz/evk-periphery/blob/995449e6e960b515869f00235fdcd417ad8e08fc/src/HookTarget/HookTargetStakeDelegator.sol#L296-L303) during the withdrawal process. As a result, if an account’s balance consistently increases and no withdrawals occur, the delegated stake will remain split between the accounts and its registered EVC owner, rather than being fully migrated to the owner.
The issue is classified as **Medium** severity because it reduced effectiveness of the staking system.
<br/>
##### Recommendation
We recommend modifying the `checkVaultStatus()` logic to ensure the stake migration also occurs during balance increases, not only during withdrawals.

> **Client's Commentary:**
> Fixed: https://github.com/euler-xyz/evk-periphery/pull/274/files
> MixBytes: There is an incorrect accounting for the delegated stake. The issue is described in the H-1 above.

---
### Example 3

**Auto Label:** Failure to properly validate or update ownership and state during stake delegation and migration leads to unclaimed rewards, double-accounting, and inconsistent stake balances.  

**Original Text Preview:**

##### Description
There is an issue with [how `_migrateStake` is used](https://github.com/euler-xyz/evk-periphery/blob/647866626fbceec678e3e11cdd9d7e5be9e5f6e5/src/HookTarget/HookTargetStakeDelegator.sol#L283) inside the `_delegateStake` function. `_migrateStake` internally delegates the stake from the account to the owner and returns the delegated amount. This amount is then added to the `amount` value originally passed to the `_delegateStake` function.

The issue is classified as **High** severity because it leads to double-accounting for the delegated value from the account to the owner address, which is incorrectly added to the original `amount`, which was subject to delegation.
<br/>
##### Recommendation
We recommend changing the logic inside the internal `_delegateStake` function to the following lines:

```solidity
function _delegateStake(
    address account, 
    uint256 amount) internal {
    address owner = evc.getAccountOwner(account);
    
    _migrateStake(owner, account);
    rewardVault.delegateStake(
        owner == address(0) ? account : owner, 
        amount);
}
```

This ensures that the migrated stake is not double-counted, and there is a maintained clear distinction between migration and new delegation.

> **Client's Commentary:**
> Fixed: https://github.com/euler-xyz/evk-periphery/pull/270/files

---

---
