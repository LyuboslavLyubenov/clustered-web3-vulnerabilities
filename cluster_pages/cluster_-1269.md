# Cluster -1269

**Rank:** #186  
**Count:** 64  

## Label
Reentrancy attacks exploiting ERC777 token hooks, enabling attackers to drain funds, manipulate loan states, or cause denial-of-service through unauthorized token transfers and state inconsistencies.

## Cluster Information
- **Total Findings:** 64

## Examples

### Example 1

**Auto Label:** Reentrancy attacks exploiting ERC777 token hooks, enabling attackers to drain funds, manipulate loan states, or cause denial-of-service through unauthorized token transfers and state inconsistencies.  

**Original Text Preview:**

## Vulnerability Report

## Severity: Low Risk

### Context
(No context files were provided by the reviewers)

### Description
In the current implementation of Provisioner, there is a potential reentrancy vector between `solveRequestsDirect` (via `_solveDepositDirect`) and `refundRequest`, as seen in `Provisioner.sol#L304-L327` and `L230-L258`.

The `asyncDepositHashes` mapping is updated only after transferring tokens during `solveRequestsDirect`, which means that during a callback (e.g., from a malicious token contract), the attacker can call `refundRequest`. This results in the token being transferred again from the Provisioner, causing a double withdrawal of the same deposit.

To construct this attack:
- The attacker first creates a deposit request for a stale or invalid price that will not be solved.
- When calling `solveRequestsDirect`, the vault attempts to execute the deposit, transferring tokens to the vault.
- During this transfer, the attacker re-enters and calls `refundRequest`, which executes before the `asyncDepositHashes` entry is cleared.
- As a result, both functions complete and transfer the same tokens, enabling the attacker to extract twice their deposit.

While this scenario is highly unlikely today due to several constraints (non-acceptance of tokens with callback behavior, no support for ERC777-like tokens, no native currency support yet), it remains an architectural hazard if native currency or callback-based tokens are ever supported.

### Recommendation
To follow CEI best practices and prevent this class of issue:
- Move the clearing of `asyncDepositHashes` earlier in `_solveDepositDirect`, before any external token transfers.
- Consider doing the same in `_solveRedeemDirect` to future-proof symmetric flows.
- Optionally apply `nonReentrant` modifiers on user-facing functions like `solveRequestsDirect` and `refundRequest`.

These changes would align the functions with reentrancy-safe patterns already followed in other parts of the codebase.

### Aera
Fixed in PR 346.

### Spearbit
Fix verified.

---
### Example 2

**Auto Label:** Reentrancy attacks exploiting ERC777 token hooks, enabling attackers to drain funds, manipulate loan states, or cause denial-of-service through unauthorized token transfers and state inconsistencies.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-01-peapods-finance-judging/issues/37 

## Found by 
0xgh0st, X77, gegul

### Summary

`PodUnwrapLocker` can be drained due to an arbitrary input

### Root Cause

`PodUnwrapLocker` provides a functionality to debond your pod tokens without paying a fee by locking them for a period of time. The issue is that the function allows an arbitrary pod input which allows for a complete drain of the contract:
```solidity
function debondAndLock(address _pod, uint256 _amount) external nonReentrant {
       // NO CHECK FOR THE POD HERE
}
```

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. A malicious contract is provided as the pod upon calling `debondAndLock()`
2. In the line below, pod returns an array of 2 addresses which are both the same token, token `X` (token to be drained, deposited by other users who are using the contract functionality):
```solidity
IDecentralizedIndex.IndexAssetInfo[] memory _podTokens = _podContract.getAllAssets();
```
3. In the line below, we store the token `X` balances in both of the array fields, let's say 100 tokens:
```solidity
for (uint256 i = 0; i < _tokens.length; i++) {
            _tokens[i] = _podTokens[i].token;
            _balancesBefore[i] = IERC20(_tokens[i]).balanceOf(address(this));
        }
```
4. We then call `debond()` on our contract which transfers in 100 tokens of `X`
5. In the line below, we fill the received amounts array with 100 tokens twice:
```solidity
for (uint256 i = 0; i < _tokens.length; i++) {
            _receivedAmounts[i] = IERC20(_tokens[i]).balanceOf(address(this)) - _balancesBefore[i];
        }
```
6. We return our own config with a 0 duration to immediately withdraw and store our received amounts in the locks mapping
7. We go over our token `X` twice with 100 amount in both fields, receiving 200 tokens even though we put in 100

### Impact

Theft of funds

### PoC

_No response_

### Mitigation

Do not allow arbitrary pods

---
### Example 3

**Auto Label:** Arbitrary control and unauthorized access leading to indefinite lockups, reward hijacking, and reentrancy-based state manipulation, compromising user asset access and reward integrity.  

**Original Text Preview:**

Here is the order of operations:

```solidity
_claimRewards(noteId, stakeInfo, recipient);

positionManager.safeTransferFrom(address(this), msg.sender, stakeInfo.tokenId);

delete stakes[noteId];
```

`safeTransferFrom()` will call the `onERC721Received()` function on `msg.sender` if `msg.sender` is a contract.
At this point, since `stakes[noteId]` has not yet been deleted, the user can transfer the NFT back to the `UniswapV3Staking` contract, and then re-enter `unstake()` once again and it will not revert.

Since `_claimRewards()` updates `stakeInfo.lastRewardTime` to `block.timestamp`, the rewards earned upon reentrancy will be `0`, so the reentrancy does not cause any fund loss. However, it is risky because any future changes to the reward calculation can potentially lead to a critical vulnerability where an attacker re-enters unstake() to drain the rewards.

Currently the only impact is that the `Unstaked` event can be spammed an unlimited amount of times via this reentrancy.

The recomendation is to change the order of operations as follows:

```diff
_claimRewards(noteId, stakeInfo, recipient);

-positionManager.safeTransferFrom(address(this), msg.sender, stakeInfo.tokenId);

delete stakes[noteId];

+positionManager.safeTransferFrom(address(this), msg.sender, stakeInfo.tokenId);

```

---
