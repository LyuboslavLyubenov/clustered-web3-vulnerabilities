# Cluster -1098

**Rank:** #189  
**Count:** 60  

## Label
Inconsistent sender validation via `msg.sender` instead of `_msgSender()` enables spoofed or unauthorized access, leading to privilege escalation, unauthorized operations, and denial-of-service in proxy or meta-transaction environments.

## Cluster Information
- **Total Findings:** 60

## Examples

### Example 1

**Auto Label:** Inconsistent sender validation via `msg.sender` instead of `_msgSender()` enables spoofed or unauthorized access, leading to privilege escalation, unauthorized operations, and denial-of-service in proxy or meta-transaction environments.  

**Original Text Preview:**

## Severity

High Risk

## Description

The `DaosLocker::collect()` function is vulnerable to impersonation attacks. A malicious actor can deploy an **arbitrary contract** that mimics the **token address and LP token ID** of a legitimate `DaosLive` contract. By doing so, they can illegitimately invoke `collect()` and **claim all the accrued swap fees**, effectively **stealing earnings meant for the rightful owner**.

## Exploit Scenario

1. A malicious actor observes the finalized `DaosLive` contract.
2. They deploy a **fake contract** that uses the **same token address and LP token ID** as the original `DaosLive` contract.
3. The attacker then calls `DaosLocker::collect()` using the malicious contract.
4. The function executes and **transfers the Dao fees** to the attacker — fees that were originally intended for the legitimate owner of the `DaosLive` contract.

### Root Cause

- Lack of **authorization checks** on the caller of `collect()`
- Absence of **contract ownership or identity verification**

## Location of Affected Code

File: [contracts/DaosLocker.sol](https://github.com/ED3N-Ventures/daoslive-sc/blob/9a1856db2060b609a17b24aa72ab35f2cdf09031/contracts/DaosLocker.sol)

```solidity
function collect(address dao) external {
  // code
  IDaosLive daosLive = IDaosLive(dao);
  @>  address token = daosLive.token();
  INonfungiblePositionManager positionManager = INonfungiblePositionManager(
        _factory.uniV3PositionManager()
  );
  address wethAddr = positionManager.WETH9();

  (uint256 amount0, uint256 amount1) = positionManager.collect(
      INonfungiblePositionManager.CollectParams({
          recipient: address(this),
          amount0Max: type(uint128).max,
          amount1Max: type(uint128).max,
   @>     tokenId: daosLive.lpTokenId()
      })
  );

  TransferHelper.safeTransfer(
        wethAddr,
  @>    OwnableUpgradeable(dao).owner(),
            daoEth
        );
        TransferHelper.safeTransfer(
            token,
        @>  OwnableUpgradeable(dao).owner(),
            daoToken
        );
  // code
}
```

## Impact

**Drain all accumulated Uniswap swap fees** that were meant for legitimate DAOs.
**Completely bypass the intended ownership model**, allowing external actors to claim protocol rewards.

## Recommendation

Consider implementing the steps described below to mitigate this issue:

- Create a function inside `DaosLive` to call `DaosLocker::collect()`
- Instead of taking input, use `msg.sender` for Dao address

## Team Response

Fixed.

---
### Example 2

**Auto Label:** Misuse of `msg.sender` in multicall contexts leads to incorrect sender identity, enabling unauthorized access, logic errors, and MEV exploitation due to failure to track original transaction originators.  

**Original Text Preview:**

**Description:** `LibMulticaller` is used throughout the codebase to retrieve the actual sender of multicall transactions; however, `BunniToken::_beforeTokenTransfer` and `BunniToken::_afterTokenTransfer` both incorrectly pass `msg.sender` directly to the corresponding Hooklet function:

```solidity
function _beforeTokenTransfer(address from, address to, uint256 amount, address newReferrer) internal override {
    IHooklet hooklet_ = hooklet();
    if (hooklet_.hasPermission(HookletLib.BEFORE_TRANSFER_FLAG)) {
        hooklet_.hookletBeforeTransfer(msg.sender, poolKey(), this, from, to, amount);
    }
}

function _afterTokenTransfer(address from, address to, uint256 amount) internal override {
    // call hooklet
    IHooklet hooklet_ = hooklet();
    if (hooklet_.hasPermission(HookletLib.AFTER_TRANSFER_FLAG)) {
        hooklet_.hookletAfterTransfer(msg.sender, poolKey(), this, from, to, amount);
    }
}
```

**Impact:** The Hooklet calls will reference the incorrect sender. This has potentially serious downstream effects for integrators as custom logic is executed with the incorrect address in multicall transactions.

**Recommended Mitigation:** `LibMulticaller.senderOrSigner()` should be used in place of `msg.sender` wherever the actual sender is required:

```diff
function _beforeTokenTransfer(address from, address to, uint256 amount, address newReferrer) internal override {
    IHooklet hooklet_ = hooklet();
    if (hooklet_.hasPermission(HookletLib.BEFORE_TRANSFER_FLAG)) {
--      hooklet_.hookletBeforeTransfer(msg.sender, poolKey(), this, from, to, amount);
++      hooklet_.hookletBeforeTransfer(LibMulticaller.senderOrSigner(), poolKey(), this, from, to, amount);
    }
}

function _afterTokenTransfer(address from, address to, uint256 amount) internal override {
    // call hooklet
    IHooklet hooklet_ = hooklet();
    if (hooklet_.hasPermission(HookletLib.AFTER_TRANSFER_FLAG)) {
--      hooklet_.hookletAfterTransfer(msg.sender, poolKey(), this, from, to, amount);
++      hooklet_.hookletAfterTransfer(LibMulticaller.senderOrSigner(), poolKey(), this, from, to, amount);
    }
}
```

**Bacon Labs:** Fixed in [PR \#106](https://github.com/timeless-fi/bunni-v2/pull/106).

**Cyfrin:** Verified, the `LibMulticaller` is now used to pass the `msg.sender` in `BunniToken` Hooklet calls.

---
### Example 3

**Auto Label:** Inconsistent sender validation via `msg.sender` instead of `_msgSender()` enables spoofed or unauthorized access, leading to privilege escalation, unauthorized operations, and denial-of-service in proxy or meta-transaction environments.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-04-zetachain-cross-chain-judging/issues/312 

## Found by 
0x15, 0xkmr\_, OhmOudits, berndartmueller, ch13fd357r0y3r, shaflow01

## Summary

The `sender` argument for the Solana `OnCall` instruction can be impersonated due to not being included in the message signed by the observers. This allows an attacker to call the destination program with a different `sender` address than the one originally initiated the CCTX.

## Root Cause

The `sender` argument for [`handle_spl_token()`](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/protocol-contracts-solana/programs/gateway/src/instructions/execute.rs#L100) and [`handle_sol()`](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/protocol-contracts-solana/programs/gateway/src/instructions/execute.rs#L40) is provided to the called destination program as part of the `OnCall` instruction. For example, authorization checks can be implemented to verify the original CCTX sender.

However, the `sender` argument is not included in the message signed by the observers. This means the `sender` can be exchanged to any other address when calling the gateway's `execute()` or `execute_spl_token()` function. As a result, the `sender` can be impersonated by an attacker, allowing them to call the destination program with a different `sender` address than the one originally initiated the CCTX.

As Solana does not have a public mempool, exploiting these issues is difficult. However, a single observer could exploit this by frontrunning the outbound Solana transaction and replacing the `sender`. Or, the Solana RPC operator or the validator could also exploit it.

## Internal Pre-conditions

- A single malicious observer

## External Pre-conditions

- The destination Solana program has authorization checks that depend on the `sender` argument and distribute funds based on it.

## Attack Path

1. Attacker sends an outbound CCTX from ZetaChain to Solana to call a specific destination program. The attacker's address will be used as the CCTX sender.
2. Observers vote on the outbound CCTX.
3. The observer colludes with the attacker and replaces the `sender` (the attacker's address) with their target address that should be impersonated.
4. The Solana transaction is executed, and the destination program is called with the impersonated `sender` address.
5. Destination program checks the `sender` address and authorizes funds transfer to the impersonated address.

## Impact

The `sender` argument for the `OnCall` instruction can be impersonated.

## PoC

N/A

## Mitigation

The `sender` argument should be included in the message signed by the observers.


## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/zeta-chain/node/pull/3896


**gjaldon**

The issue has been fixed by [including](https://github.com/zeta-chain/node/pull/3896/files\#diff-367bfa6b34b8ec2e1f482eafb11e80a929d53795c766224f7b36df40396a3c1fR342-R346) the CCTX sender in the message hash.

---
