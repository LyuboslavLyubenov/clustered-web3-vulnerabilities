# Cluster -1113

**Rank:** #106  
**Count:** 159  

## Label
Insufficient input validation enables attackers to manipulate profit, reward, or liquidity calculations by injecting forged or invalid data, leading to unauthorized fund drainage, reward theft, or profit minimization.

## Cluster Information
- **Total Findings:** 159

## Examples

### Example 1

**Auto Label:** Insufficient input validation leads to unauthorized operations, incorrect routing, or unintended token transfers, enabling denial-of-service, fund loss, or malicious redirection.  

**Original Text Preview:**

The [`SwapProxy` contract](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L26) contains the [`performSwap` function](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L67), which allows the caller to execute a swap in two ways: [by approving or sending tokens to the specified exchange](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L81-L84), or by [approving tokens through the `Permit2` contract](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L86-L101). However, since it is possible to supply any address as the `exchange` parameter and any call data through the `routerCalldata` parameter of the `performSwap` function, the `SwapProxy` contract may be forced to perform an [arbitrary call to an arbitrary address](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L108).

This could be exploited by an attacker, who could force the `SwapProxy` contract to call the [`invalidateNonces` function](https://github.com/Uniswap/permit2/blob/cc56ad0f3439c502c246fc5cfcc3db92bb8b7219/src/AllowanceTransfer.sol#L113) of the `Permit2` contract, specifying an arbitrary spender and a nonce higher than the current one. As a result, the nonce for the given (token, spender) pair will be [updated](https://github.com/Uniswap/permit2/blob/cc56ad0f3439c502c246fc5cfcc3db92bb8b7219/src/AllowanceTransfer.sol#L124). If the `performSwap` function is called again later, it [will attempt to use a subsequent nonce](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L95), which has been invalidated by the attacker and the code inside `Permit2` will [revert due to nonces mismatch](https://github.com/Uniswap/permit2/blob/cc56ad0f3439c502c246fc5cfcc3db92bb8b7219/src/AllowanceTransfer.sol#L138).

As the `performSwap` function is the only place where the nonce passed to the `Permit2` contract is updated, the possibility of swapping a given token on a certain exchange will be blocked forever, which impacts all the functions of the [`SpokePoolPeriphery` contract](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L140) related to swapping tokens. The attack may be performed for many different (tokens, exchange) pairs.

Consider not allowing the `exchange` parameter to be equal to the `Permit2` contract address.

***Update:** Resolved in [pull request #1016](https://github.com/across-protocol/contracts/pull/1016) at commit [`713e76b`](https://github.com/across-protocol/contracts/pull/1016/commits/713e76b8388d90b4c3fbbe3d16b531d3ef81c722).*

---
### Example 2

**Auto Label:** Insufficient input validation leads to unauthorized operations, incorrect routing, or unintended token transfers, enabling denial-of-service, fund loss, or malicious redirection.  

**Original Text Preview:**

`swapActive()` is called by critical flows:

1. `stateChangeActive()` --> `swapActive()`.
2. `sync()` --> `_poke()` --> `_closeTrustedFill()` --> `closeFiller()` --> `swapActive()`.

**Case1:**
External integrating protocol calls `stateChangeActive()` and wants to see `(false, false)` before trusting the Folio state. Suppose:

- `sellAmount` is `100`.
- `sellTokenBalance` is `98` i.e. swap is still active and only partial sell has occurred.
- Ideally external protocol should see `(false, true)`.
- Attack scenario: Attacker front runs the call to `stateChangeActive()` and donates `2` sellTokens.
- Now, inside the `swapActive()` function, `if (sellTokenBalance >= sellAmount)` evaluates to `true`, and swap is reported as completed i.e. `false` is returned.
- External protocol trusts an inconsistent state & balance.

Note that a similar attack can be mounted by donating buyTokens which would result in `minimumExpectedIn > buyToken.balanceOf(address(this))` to return `false` instead of `true`.

**Case2:**
Ideally all `sync()` modifier protected functions should revert if swaps are active because `closeFiller()` checks that:

```js
require(!swapActive(), BaseTrustedFiller__SwapActive());
```

Almost all the critical Folio functions like `mint()`, `redeem()`, `distributeFees()` etc are protected by `sync()`. But due to the same attack path as above, protocol can be fooled into believing that the swap has concluded. This allows the function calls to proceed and potentially end up in an inconsistent state.

**Recommendations:**

Track balance internally through accounting variables instead of using `balanceOf()` inside `swapActive()`.

---
### Example 3

**Auto Label:** Lack of input validation enables attackers to manipulate critical parameters or submit forged data, leading to consensus errors, reward manipulation, or denial-of-service.  

**Original Text Preview:**

## Reports

**Severity:** Medium Risk  
**Context:** (No context files were provided by the reviewer)  
**Description:** This issue was found during the fixes review period in commit hash `e2bac5c`. As part of mitigating the issues found during the review, the Kinetiq team introduced a new contract named `ValidatorSanityChecker` which is used for validity checks on oracle reports. The contract has one main function named `checkValidatorSanity` which, among other checks, validates that given values (such as `avgUptimeScore`, `avgSpeedScore`, etc.) are inside bounds based on storage variables that can be set. The issue here is that these storage variables can be set by anyone, which can be exploited by attackers to cause the rejection of oracle reports. The full list of unprotected functions:

1. `setSlashingTolerance()`
2. `setRewardsTolerance()`
3. `setScoreTolerance()`
4. `setMaxScoreBound()`

**Recommendation:** Consider restricting access to these functions to privileged accounts only.  
**Kinetiq:** Fixed in commit `b739069`.  
**Cantina Managed:** Fix verified.

---
