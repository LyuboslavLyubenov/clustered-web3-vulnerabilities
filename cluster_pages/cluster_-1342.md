# Cluster -1342

**Rank:** #92  
**Count:** 183  

## Label
Missing account or input validation enables attackers to forge or manipulate token accounts, bypass ownership checks, and drain vaults or misrepresent balances, leading to financial loss and protocol compromise.

## Cluster Information
- **Total Findings:** 183

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

**Auto Label:** Insufficient input validation leads to unauthorized operations, incorrect routing, or unintended token transfers, enabling denial-of-service, fund loss, or malicious redirection.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-05-dodo-cross-chain-dex-judging/issues/416 

## Found by 
0xc0ffEE, AnomX, CoheeYang, Cybrid, EgisSecurity, X0sauce, edger, hunt1, pyk, silver\_eth, wellbyt3

### Summary

 Whenever [`withdrawToNativeChain`](https://github.com/sherlock-audit/2025-05-dodo-cross-chain-dex/blob/d4834a468f7dad56b007b4450397289d4f767757/omni-chain-contracts/contracts/GatewayTransferNative.sol#L530) or[`onCall`](https://github.com/sherlock-audit/2025-05-dodo-cross-chain-dex/blob/d4834a468f7dad56b007b4450397289d4f767757/omni-chain-contracts/contracts/GatewayTransferNative.sol#L395) is called on the  [`GatewayTransferNative`](https://github.com/sherlock-audit/2025-05-dodo-cross-chain-dex/blob/d4834a468f7dad56b007b4450397289d4f767757/omni-chain-contracts/contracts/GatewayTransferNative.sol#L20C10-L20C31)  . The [`_doMixSwap()`](https://github.com/sherlock-audit/2025-05-dodo-cross-chain-dex/blob/d4834a468f7dad56b007b4450397289d4f767757/omni-chain-contracts/contracts/GatewayTransferNative.sol#L425)  is trigger and the function returns the original token amount without enforcing a swap when `swapData` is empty. If `decoded.targetZRC20 != zrc20`, this leads to unexpected token transfers, allowing Attacker to bypass swaps and drain higher-value assets held by the contract.


### Root Cause

https://github.com/sherlock-audit/2025-05-dodo-cross-chain-dex/blob/d4834a468f7dad56b007b4450397289d4f767757/omni-chain-contracts/contracts/GatewayTransferNative.sol#L430
```solidity
if (swapData.length == 0) {
    return amount;
}
```

This check skips the swap entirely when `swapData` is empty, and there is no check whether the input `zrc20` == `decoded.targetZRC20`



### Internal Pre-conditions

- The caller uses `withdrawToNativeChain()` or `onCall()`  
- Sets `swapData = ""` (empty)  
- Sets `decoded.targetZRC20` to a different token than `zrc20`


### External Pre-conditions

- `decoded.targetZRC20` represents a significantly higher-value token compare to the deposited `zrc20 `(e.g., ETH.ARB ~$2300 vs AVAX ~$20)  
- The contract holds those tokens, often due to refund flows from reverted cross-chain messages


### Attack Path

1. Attacker calls `withdrawToNativeChain()` with:
   - `fromToken = AVAX-ZRC`
   - `decoded.targetZRC20 = ETH.ARB-ZRC`
   - `swapData = ""`
2. `withdrawToNativeChain()`  functions decodes the payload and calls `_doMixSwap()`
3. `_doMixSwap()` skips the swap and returns the full AVAX amount
4. User receives ETH.ARB tokens in 1:1 amount (in wei), leading to a huge gain  
5. Protocol loses high-value tokens without performing a valid swap


### Impact

This can lead to major loss of funds from the protocol’s ZRC20 token balances, especially those intended for refunds. The attacker receives assets at favorable USD rates, causing imbalance.


### PoC

**note I am using a token difference here to show the impact  because the provided test suite has some issue with price conversion**

place this in `test/GatewayTransferNative.t.sol`  and run 
`forge test --mt test_IamOTI_badSwapdataofDoMIxSwap --fork-url https://zetachain-evm.blockpi.network/v1/rpc/public`
```solidity
function test_IamOTI_badSwapdataofDoMIxSwap() public {
    // Simulate production state: preload contract with ZRC20 balances
    token1Z.mint(address(gatewayTransferNative), 2000 ether); // token1Z = AVAX-ZRC (~$20)
    token2Z.mint(address(gatewayTransferNative), 2000 ether); // token2Z = ETH.ARB-ZRC (~$2300)

    uint256 amount = 100 ether;
    uint32 dstChainId = 421614; // Arbitrum Sepolia testnet chain ID

    // Construct the cross-chain message with a mismatched targetZRC20 and empty swapData
    address targetZRC20 = address(token2Z); // target is ETH.ARB-ZRC
    bytes memory sender = abi.encodePacked(user1);
    bytes memory receiver = abi.encodePacked(user2);
    bytes memory swapDataZ = ""; // ← this triggers the early return in _doMixSwap
    bytes memory contractAddress = abi.encodePacked(address(gatewaySendB));
    bytes memory fromTokenB = abi.encodePacked(address(token2B));
    bytes memory toTokenB = abi.encodePacked(address(token2B));
    bytes memory swapDataB = "";
    bytes memory accounts = "";

    // Encode the malicious message payload
    bytes memory message = encodeMessage(
        dstChainId,
        targetZRC20,
        sender,
        receiver,
        swapDataZ,
        contractAddress,
        abi.encodePacked(fromTokenB, toTokenB, swapDataB),
        accounts
    );

    // Execute the attack simulation
    vm.startPrank(user1);
    token1Z.approve(address(gatewayTransferNative), amount);
    gatewayTransferNative.withdrawToNativeChain(address(token1Z), amount, message);
    vm.stopPrank();

    // Validate balances
    assertEq(token1Z.balanceOf(user1), initialBalance - amount);
// it is almost 1:1 token2B.balanceOf(user2) is around 98.9e18 the different is the protocol fee and the gas fee
    assertApproxEqAbs(token2B.balanceOf(user2), amount, 1.2e18); 
}
```
#### Note 
the same issue exist in `GatewayCrossChain`


### Mitigation

Add a strict validation to enforce swaps when the tokens differ:

```solidity
  function _doMixSwap(address zrc20,bytes memory swapData, uint256 amount, MixSwapParams memory params)
        internal
        returns (uint256 outputAmount)
    {
        if (swapData.length == 0) 
       require(params.toToken == zrc20 );
            return amount;
        }
___rest of the code 
}
```

## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/Skyewwww/omni-chain-contracts/pull/33

---
