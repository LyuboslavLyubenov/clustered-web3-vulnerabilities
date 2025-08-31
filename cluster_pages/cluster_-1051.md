# Cluster -1051

**Rank:** #428  
**Count:** 9  

## Label
Users bypass deposit caps by transferring funds to other addresses before depositing, exploiting flawed balance checks that ignore actual balances or recipient addresses, leading to unauthorized, unbounded deposits and fund loss.

## Cluster Information
- **Total Findings:** 9

## Examples

### Example 1

**Auto Label:** Vulnerabilities arise when deposit limits are incorrectly enforced or bypassed via gas refunds or flawed address checks, allowing users to circumvent caps and undermine intended security boundaries.  

**Original Text Preview:**

In order to bridge tokens from Ethereum to a ZkStack blockchain using a custom gas token, the [`relayTokens` function](https://github.com/across-protocol/contracts/blob/5a0c67c984d19a3bb843a4cec9bb081734583dd1/contracts/chain-adapters/ZkStack_CustomGasToken_Adapter.sol#L141) of the `ZkStack_CustomGasToken_Adapter` contract can be used. In case the token to bridge is WETH, the token is first [converted](https://github.com/across-protocol/contracts/blob/5a0c67c984d19a3bb843a4cec9bb081734583dd1/contracts/chain-adapters/ZkStack_CustomGasToken_Adapter.sol#L155) to ETH, and then that ETH is bridged [using the `requestL2TransactionTwoBridges` function](https://github.com/across-protocol/contracts/blob/5a0c67c984d19a3bb843a4cec9bb081734583dd1/contracts/chain-adapters/ZkStack_CustomGasToken_Adapter.sol#L157-L168) of the `BridgeHub` contract. The `requestL2TransactionTwoBridges` function then [calls the `bridgeHubDeposit` function of the second bridge](https://etherscan.io/address/0x509da1be24432f8804c4a9ff4a3c3f80284cdd13#code#F1#L291) with the ETH amount specified by the caller.

However, the `bridgeHubDeposit` function [requires that the deposit amount specified equals 0](https://github.com/matter-labs/era-contracts/blob/aafee035db892689df3f7afe4b89fd6467a39313/l1-contracts/contracts/bridge/L1SharedBridge.sol#L328) in case when ETH is bridged, yet it [is specified as a nonzero amount](https://github.com/across-protocol/contracts/blob/5a0c67c984d19a3bb843a4cec9bb081734583dd1/contracts/chain-adapters/ZkStack_CustomGasToken_Adapter.sol#L167) inside the `relayTokens` function of the adapter. This will cause any attempt to bridge WETH to L2 to revert.

In cases where WETH is being bridged, consider setting the amount to be used in the second bridge's calldata to 0.

***Update:** Resolved in [pull request #743](https://github.com/across-protocol/contracts/pull/743) at commit [0bdad5b](https://github.com/across-protocol/contracts/pull/743/commits/0bdad5bdaacbbaaa9c89addac9e8ce8c36e8f8d5).*

---
### Example 2

**Auto Label:** Users bypass deposit caps by transferring funds to other addresses before depositing, exploiting flawed balance checks that ignore actual balances or recipient addresses, leading to unauthorized, unbounded deposits and fund loss.  

**Original Text Preview:**

<https://github.com/code-423n4/2024-12-bakerfi/blob/main/contracts/core/VaultBase.sol# L251>

### Summary

Depositors can easily bypass the maximum deposit limit by specifying another recipient address than the one they are depositing from

### Vulnerability Details

`VaultBase::_depositInternal()` limits the amount of assets a single depositor (wallet) can deposit into the vault if `_maxDeposit > 0`. We can see the following checks being performed:
```

    function _depositInternal(uint256 assets, address receiver) private returns (uint256 shares) {
        //...

        // Check if deposit exceeds the maximum allowed per wallet
        uint256 maxDepositLocal = getMaxDeposit();
        if (maxDepositLocal > 0) {
@->            uint256 depositInAssets = (balanceOf(msg.sender) * _ONE) / tokenPerAsset();
            uint256 newBalance = assets + depositInAssets;
            if (newBalance > maxDepositLocal) revert MaxDepositReached();
        }
        //...
    }
```

The problem here is that we take the shares of `msg.sender` which can be different from the `recipient` specified when depositing. This means that a single depositor can deposit as much as he wants, he just has to specify a different recipient address.

**Attack path**

1. Bob has never deposited in the vault so his shares are 0 - `balanceOf(Bob) = 0`
2. He creates a second wallet to recieve the shares that will be minted to him on deposit
3. Bob calls `Vault::deposit()` specifying his second wallet as a recipient
4. At the end of the deposit transaction `balanceOf(Bob)` is still 0 because we mint the shares to the recipient (Bob’s second wallet)
5. Bob repeats this process as much as he wants and his `depositInAssets` will always be 0 so he can never reach the deposit limit.

### Impact

Bypassing the deposit limit

### Recommended mitigation steps

Introduce a `mapping` that keeps track of the deposits of `msg.sender`

**chefkenji (BakerFi) confirmed**

**[BakerFi mitigated](https://github.com/code-423n4/2025-01-bakerfi-mitigation?tab=readme-ov-file# findings-being-mitigated):**

> [PR-4](https://github.com/baker-fi/bakerfi-contracts/pull/4)

**Status:** Unmitigated. Full details in report from [shaflow2](https://code4rena.com/evaluate/2025-01-bakerfi-mitigation-review/findings/S-20), and also included in the [Mitigation Review](# mitigation-review) section below.

---

---
### Example 3

**Auto Label:** Users bypass deposit caps by transferring funds to other addresses before depositing, exploiting flawed balance checks that ignore actual balances or recipient addresses, leading to unauthorized, unbounded deposits and fund loss.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-12-plaza-finance-judging/issues/278 

The protocol has acknowledged this issue.

## Found by 
056Security, 0x23r0, 0xDemon, 0xShahilHussain, 0xc0ffEE, Adotsam, JohnTPark24, Kenn.eth, Kirkeelee, MysteryAuditor, Pablo, X0sauce, Xcrypt, ZeroTrust, ZoA, bladeee, dobrevaleri, farismaulana, future, hrmneffdii, makeWeb3safe, phoenixv110, super\_jack, tutiSec, y4y

### Summary

[`BalancerRouter::joinBalancerAndPredeposit`](https://github.com/sherlock-audit/2024-12-plaza-finance/blob/main/plaza-evm/src/BalancerRouter.sol#L23C1-L40C6)  calls [`Predeposit::deposit`](https://github.com/sherlock-audit/2024-12-plaza-finance/blob/main/plaza-evm/src/PreDeposit.sol#L118C1-L134C4) to deposit balancerPoolTokens into the `Predeposit` contract. In the `Predeposit::deposit` if `reserveAmount + amount > reserveCap`, the difference is deposited and the rest remains with `msg.sender`, which in this case is the `BalancerRouter`.  

The user will have less than expected amount deposited in `Predeposit` without getting refund for the rest of the funds. 

```js
      function _deposit(uint256 amount, address onBehalfOf) private checkDepositStarted checkDepositNotEnded {
        if (reserveAmount >= reserveCap) revert DepositCapReached();
    
        address recipient = onBehalfOf == address(0) ? msg.sender : onBehalfOf;
    
        // if user would like to put more than available in cap, fill the rest up to cap and add that to reserves
@>        if (reserveAmount + amount >= reserveCap) {
@>          amount = reserveCap - reserveAmount;
        }
    
        balances[recipient] += amount;
        reserveAmount += amount;
    
        IERC20(params.reserveToken).safeTransferFrom(msg.sender, address(this), amount);
    
        emit Deposited(recipient, amount);
      }
```

### Root Cause

_No response_

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

Assume `reserveCap` is `1,000 BPT` and `reserveAmount` is `900BPT` 
User calls `BalancerRouter::joinBalancerAndPredeposit` and deposits assets worth `500 BPT`
Only `100 BPT` is deposited and users loses `400 BPT` which is trapped inside the contract.


### Impact

Loss of funds

### PoC

_No response_

### Mitigation

```diff
        function joinBalancerAndPredeposit(
            bytes32 balancerPoolId,
            address _predeposit,
            IAsset[] memory assets,
            uint256[] memory maxAmountsIn,
            bytes memory userData
        ) external nonReentrant returns (uint256) {
            // Step 1: Join Balancer Pool
            uint256 balancerPoolTokenReceived = joinBalancerPool(balancerPoolId, assets, maxAmountsIn, userData);
    
            // Step 2: Approve balancerPoolToken for PreDeposit
            balancerPoolToken.safeIncreaseAllowance(_predeposit, balancerPoolTokenReceived);
+            uint256 BPTBalanceBefore = balancerPoolToken.balanceOf(address(this));

            // Step 3: Deposit to PreDeposit
            PreDeposit(_predeposit).deposit(balancerPoolTokenReceived, msg.sender);
+         uint256 BPTBalanceAfter = balancerPoolToken.balanceOf(address(this));
+          if (BPTBalanceAfter - BPTBalanceBefore > 0) {
+             revert ( 'ReserveCap Reached` ); }
            return balancerPoolTokenReceived;
        }
```

---
