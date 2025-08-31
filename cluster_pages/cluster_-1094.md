# Cluster -1094

**Rank:** #95  
**Count:** 173  

## Label
Replay and state overwrite attacks via insufficient input validation and deterministic identifier generation, enabling unauthorized access, transaction manipulation, and irreversible recovery failures.

## Cluster Information
- **Total Findings:** 173

## Examples

### Example 1

**Auto Label:** Replay attacks via missing nonce checks or incomplete state inclusion in signatures, enabling signature reuse across transactions and compromising transaction integrity and order.  

**Original Text Preview:**

The [`SpokePoolPeriphery` contract](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L140) allows users to deposit or swap-and-deposit tokens into a SpokePool. In order to do that, the assets are first transferred from the depositor's account, optionally swapped to a different token, and then finally deposited into a SpokePool.

Assets can be transferred from the depositor's account in several different ways, including approval followed by the [`transferFrom` call](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L236-L240), [approval through the ERC-2612 `permit` function followed by `transferFrom`](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L267-L268), [transfer through the `Permit2` contract](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L295-L302), and [transfer through the ERC-3009 `receiveWithAuthorization` function](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L328-L338). The last three methods require additional user signatures and may be executed by anyone on behalf of a given user. However, the [data to be signed for deposits or swaps and deposits](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/interfaces/SpokePoolPeripheryInterface.sol#L70-L105) with ERC-2612 `permit` and with ERC-3009 `receiveWithAuthorization` does not contain a nonce, and, as such, the signatures used for these methods once can be replayed later.

The attack can be performed if a victim signs data for a function relying on the ERC-2612 `permit` function and wants to deposit tokens once again using the same method and token [within the time window determined by the `depositQuoteTimeBuffer` parameter](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePool.sol#L1306-L1307). In such a case, an attacker can first approve tokens on behalf of the victim and then call the [`swapAndBridgeWithPermit` function](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L249) or the [`depositWithPermit` function](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L357), providing a signature for a deposit or swap-and-deposit from the past, that includes fewer tokens than the approved amount.

As a result, the tokens will be deposited and potentially swapped, using the data from an old signature, forcing the victim to either perform an unintended swap or bridge the tokens to a different chain than intended. Furthermore, since the attack consumes some part of the `permit` approval, it will not be possible to deposit tokens on behalf of a depositor using the new signature until the full amount of tokens is approved by them once again. A similar attack is also possible in the case of functions that rely on the ERC-3009 `receiveWithAuthorization` function, but it requires the amount of tokens being transferred to be identical to the amount from the past.

Consider adding a nonce field into the [`SwapAndDepositData` and `DepositData` structs](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/interfaces/SpokePoolPeripheryInterface.sol#L70-L105) and storing a nonce for each user in the `SpokePoolPeriphery` contract, which should be incremented when a signature is verified and accepted.

***Update:** Resolved in [pull request #1015](https://github.com/across-protocol/contracts/pull/1015). The Across team has added a `permitNonces` mapping and extended both `SwapAndDepositData` and `DepositData` with a `nonce` field. In `swapAndBridgeWithPermit` and `depositWithPermit`, the contract now calls `_validateAndIncrementNonce(signatureOwner, nonce)` before verifying the EIP-712 signature, ensuring each permit-based operation can only be executed once. ERC-3009 paths continue to rely on the token’s own nonce; a replay here would require a token to implement both ERC-2612 and ERC-3009, a user to reuse the exact same nonce in both signatures, and both are executed within the narrow `fillDeadlineBuffer`. Given the unlikely convergence of these conditions, the risk is negligible in practice.*

---
### Example 2

**Auto Label:** Replay attacks via missing nonce checks or incomplete state inclusion in signatures, enabling signature reuse across transactions and compromising transaction integrity and order.  

**Original Text Preview:**

The [`performSwap` function](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L67) of the `SwapProxy` contract allows for providing tokens for a swap to a specified exchange using several different methods. In particular, it allows for approving tokens for the swap [through the `Permit2` contract](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L86-L101). In order to do that, it [approves the given token amount to the `Permit2` contract](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L86) and [calls the `permit` function of the `Permit2` contract](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L88-L101).

However, the [nonce specified for that call](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L95) is global for the entire contract, whereas the `Permit2` contract stores [a separate nonce for each (owner, token, spender) tuple](https://github.com/Uniswap/permit2/blob/cc56ad0f3439c502c246fc5cfcc3db92bb8b7219/src/interfaces/IAllowanceTransfer.sol#L85). As a result, any attempt to use a different (token, spender) pair from the ones used in the first `performSwap` function call will result in the [revert](https://github.com/Uniswap/permit2/blob/cc56ad0f3439c502c246fc5cfcc3db92bb8b7219/src/AllowanceTransfer.sol#L138) due to nonce mismatch.

Consider storing and using separate nonces for each (token, spender) pair in the `SwapProxy` contract.

***Update:** Resolved in [pull request #1013](https://github.com/across-protocol/contracts/pull/1013) at commit [`3cd99c4`](https://github.com/across-protocol/contracts/pull/1013/commits/3cd99c4c9362f743a233c0747bc7829e604edfa8).*

---
### Example 3

**Auto Label:** Replay attacks via missing nonce checks or incomplete state inclusion in signatures, enabling signature reuse across transactions and compromising transaction integrity and order.  

**Original Text Preview:**

The `removeLiquidityWithPermit()` and `removeLiquidityETHWithPermit()` functions in RouterV2 are vulnerable to front-running attacks that consume user permit signatures, causing legitimate liquidity removal transactions to revert and resulting in gas fee losses and a DOS.

### Finding description

The RouterV2 contract implements ERC-2612 permit functionality without protection against front-running attacks. The vulnerability stems from the deterministic nature of permit signatures and the lack of error handling when permit calls fail.

This vulnerability follows a well-documented attack pattern that has been extensively researched. According to Trust Security’s comprehensive disclosure in January 2024: <https://www.trust-security.xyz/post/permission-denied>

> “Consider, though, a situation where permit() is part of a contract call:
>
> 
```

> function deposit(uint256 amount, bytes calldata _signature) external {
>     // This will revert if permit() was front-run
>     token.permit(msg.sender, address(this), amount, deadline, v, r, s);
>     // User loses this functionality when permit fails
>     stakingContract.deposit(msg.sender, amount);
> }
> 
```

>
> This function deposits in a staking contract on behalf of the user. But what if an attacker extracts the \_signature parameters from the `deposit()` call and frontruns it with a direct `permit()`? In this case, the end result is harmful, since the user loses the functionality that follows the `permit()`.”

In RouterV2’s `removeLiquidityWithPermit()` follows this exact vulnerable pattern:

<https://github.com/code-423n4/2025-05-blackhole/blob/92fff849d3b266e609e6d63478c4164d9f608e91/contracts/RouterV2.sol# L475-L475>
```

IBaseV1Pair(pair).permit(msg.sender, address(this), value, deadline, v, r, s);
// User loses liquidity removal functionality when permit fails
(amountA, amountB) = removeLiquidity(tokenA, tokenB, stable, liquidity, amountAMin, amountBMin, to, deadline);
```

As Trust Security formally identified: **“In fact, any function call that unconditionally performs `permit()` can be forced to revert this way.”**

The root cause of the issue within the blackhole protocol is that RouterV2 makes unprotected external calls to LP token permit functions without any fallback mechanism or error handling:
```

// Line 475 - No error handling
IBaseV1Pair(pair).permit(msg.sender, address(this), value, deadline, v, r, s);
```

This creates a systematic vulnerability where any permit-based transaction can be griefed by extracting and front-running the permit signature.

### Code Location

* `RouterV2.sol# L475` - `removeLiquidityWithPermit()` function
* `RouterV2.sol# L493` - `removeLiquidityETHWithPermit()` function

<https://github.com/code-423n4/2025-05-blackhole/blob/92fff849d3b266e609e6d63478c4164d9f608e91/contracts/RouterV2.sol# L475-L475>

<https://github.com/code-423n4/2025-05-blackhole/blob/92fff849d3b266e609e6d63478c4164d9f608e91/contracts/RouterV2.sol# L493>

### Impact

This vulnerability enables attackers to systematically deny service to users attempting to remove liquidity through permit-based functions, resulting in direct financial losses and protocol dysfunction.

* Users lose transaction fees (typically `$5-50` per transaction depending on network congestion) when their transactions revert.
* Users must pay additional gas for traditional approve+remove patterns.
* Liquidity removal becomes unreliable, forcing users to abandon efficient permit-based operations.

**Attack Prerequisites:**

* Minimal: Attacker needs only basic mempool monitoring capability and gas fees.
* No special permissions or large capital requirements.
* Attack can be automated and executed repeatedly.
* Works against any user attempting permit-based operations.

### Recommended mitigation steps

Implement a different method of permit handling by wrapping the permit call in a try-catch block and only reverting if both the permit fails and the current allowance is insufficient for the operation.
```

function removeLiquidityWithPermit(
    address tokenA,
    address tokenB,
    bool stable,
    uint liquidity,
    uint amountAMin,
    uint amountBMin,
    address to,
    uint deadline,
    bool approveMax,
    uint8 v,
    bytes32 r,
    bytes32 s
) external virtual override returns (uint amountA, uint amountB) {
    address pair = pairFor(tokenA, tokenB, stable);
    uint value = approveMax ? type(uint).max : liquidity;

    // Try permit, but don't revert if it fails
    try IBaseV1Pair(pair).permit(msg.sender, address(this), value, deadline, v, r, s) {
        // Permit succeeded
    } catch {
        // Permit failed, check if we have sufficient allowance
        require(
            IBaseV1Pair(pair).allowance(msg.sender, address(this)) >= liquidity,
            "Insufficient allowance"
        );
    }

    (amountA, amountB) = removeLiquidity(tokenA, tokenB, stable, liquidity, amountAMin, amountBMin, to, deadline);
}
```

This approach follows the industry-standard mitigation pattern that successfully resolved this vulnerability across 100+ affected codebases.

### Proof of Concept

Theoretical attack walkthrough:

1. **Setup Phase**:

   * Alice holds 1000 LP tokens in pair `0xABC...`
   * Alice wants to remove liquidity efficiently using `removeLiquidityWithPermit()`
   * Bob (attacker) monitors the mempool for permit-based transactions
2. **Signature Creation**:

   * Alice creates permit signature: `permit(alice, routerV2, 1000, deadline, nonce=5)`
   * Signature parameters: `v=27, r=0x123..., s=0x456...`
3. **Transaction Submission**:

   * Alice submits: `removeLiquidityWithPermit(tokenA, tokenB, false, 1000, 950, 950, alice, deadline, false, 27, 0x123..., 0x456...)`
   * Transaction enters mempool with 20 gwei gas price
4. **Front-Running Execution**:

   * Bob extracts parameters from Alice’s pending transaction
   * Bob submits direct call: `IBaseV1Pair(0xABC...).permit(alice, routerV2, 1000, deadline, 27, 0x123..., 0x456...)` with 25 gwei gas
   * Bob’s transaction mines first, consuming Alice’s nonce (nonce becomes 6)
5. **Victim Transaction Failure**:

   * Alice’s transaction executes but fails at line 475
   * RouterV2 calls `permit()` with already-used signature
   * LP token rejects due to invalid nonce (expects 6, gets signature for nonce 5)
   * Entire transaction reverts with “Invalid signature” or similar error
6. **Attack Result**:

   * Alice loses ~`$15` in gas fees (failed transaction cost)
   * Alice cannot remove liquidity via permit method
   * Bob spent ~`$3` in gas to grief Alice
   * Attack can be repeated indefinitely against any permit user

**Impact Example:**

* During high network activity (50+ gwei), failed transactions cost `$25-75` each
* Systematic attacks against 100 users = `$2,500-7,500` in direct user losses
* No recourse for affected users; losses are permanent

**[Blackhole mitigated](https://github.com/code-423n4/2025-06-blackhole-mitigation?tab=readme-ov-file# mitigation-of-high--medium-severity-issues)**

**Status:** Unmitigated. Full details in reports from [rayss](https://code4rena.com/audits/2025-06-blackhole-mitigation-review/submissions/S-34) and [lonelybones](https://code4rena.com/audits/2025-06-blackhole-mitigation-review/submissions/S-11), and also included in the [Mitigation Review](# mitigation-review) section below.

---

---
