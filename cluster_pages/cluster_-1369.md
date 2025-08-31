# Cluster -1369

**Rank:** #445  
**Count:** 8  

## Label
Direct ERC20 function calls without return value validation lead to silent failures, fund loss, or state corruption due to unhandled errors and non-standard token behaviors.

## Cluster Information
- **Total Findings:** 8

## Examples

### Example 1

**Auto Label:** Failure to validate ERC20 transfer return values leads to silent failures, incorrect state, and potential loss of funds or denial-of-service due to improper error handling.  

**Original Text Preview:**

There are multiple occassions in the `staking.sol` contract where the `IERC20::transferFrom` and `IERC20::transfer` functions are used to transfer the `tokens` between `msg.sender` and contract itself.

But the issue is if the `token` being transferred does not return `bool` the above transfer transaction will fail since the `IERC20` implementation expects a `bool` value as the return value.

Since the `coin flip` game expects to work with multiple tokens whitelisted by the owner, this could be an issue.

Hence this will break the core functionalities of the game when such tokens are used in the game.

Hence it is recommended to use the `safeTransfer` and `safeTransferFrom` from the openzeppelin `SafeERC20` library.

---
### Example 2

**Auto Label:** Direct ERC20 function calls without return value validation lead to silent failures, fund loss, or state corruption due to unhandled errors and non-standard token behaviors.  

**Original Text Preview:**

The ULTI contract uses `IERC20.approve` and `IERC20.transferFrom` to interact with input tokens. However, there are some tokens that don’t follow the IERC20 interface, such as USDT. This could lead to transaction reverts when deploying a contract, launching a new pool, or depositing tokens.

It's recommended to use functions such as forceApprove and safeTransferFrom from the SafeERC20 library from OpenZeppelin to interact with input tokens.

---
### Example 3

**Auto Label:** Failure to validate ERC20 transfer return values leads to silent failures, incorrect state, and potential loss of funds or denial-of-service due to improper error handling.  

**Original Text Preview:**

## Severity

Medium Risk

## Description

The `RewardsClaimer.sol` contract manages the distribution and claiming of rewards. In several places in the contract, the `transfer()` function is used when reward tokens are to be transferred, which does not check the returned value.

```solidity
IERC20(rewardToken).transfer(msg.sender, _pendingRewards);
IERC20(rewardToken).transferFrom(
msg.sender, address(this), _totalAmount
IERC20(rewardToken).transfer(_to, _amount);
IERC20(_token).transfer(_to, _amount);
```

The `ERC20.transfer()` and `ERC20.transferFrom()` functions return a bool value indicating success. This parameter needs to be checked for success.

## Impact

If the return value is not checked, the transfer may not be successful. However, the functions in `RewardsClaimer.sol` will still be considered a successful transfer.

## Location of Affected Code

File: [src/rewards/RewardsClaimer.sol](https://github.com/pear-protocol/gmx-v2-core/blob/f3329d0474013d60d183a5773093b94a9e55caae/src/rewards/RewardsClaimer.sol)

## Recommendation

It is recommended to use OpenZeppelin's SafeERC20 versions with the `safeTransfer()` and `safeTransferFrom()` functions that handle the return value check.

## Team Response

Fixed as proposed.

---
