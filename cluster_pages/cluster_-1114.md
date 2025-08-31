# Cluster -1114

**Rank:** #158  
**Count:** 84  

## Label
Failure to validate access or collect fees during asset transfers leads to unauthorized movement of tokens or value leakage, enabling attackers to withdraw assets or manipulate slashing without proper authorization or fee collection.

## Cluster Information
- **Total Findings:** 84

## Examples

### Example 1

**Auto Label:** Failure to validate access or collect fees during asset transfers leads to unauthorized movement of tokens or value leakage, enabling attackers to withdraw assets or manipulate slashing without proper authorization or fee collection.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-05-dodo-cross-chain-dex-judging/issues/700 

## Found by 
0xc0ffEE, Goran, IvanFitro, SarveshLimaye

### Summary

The `withdrawToNativeChain` function of `GatewayTransferNative` fails to collect platform fees when users bridge native asset. While the function is designed to accept native asset payments through its payable modifier, the fee collection mechanism silently fails due to a low-level call to a non-contract address. This allows users to bridge native asset while completely bypassing the platform's fee collection, resulting in lost revenue for the protocol.

### Root Cause

User will provide `_ETH_ADDRESS_` as `zrc20` param in [withdrawToNativeChain](https://github.com/sherlock-audit/2025-05-dodo-cross-chain-dex/blob/main/omni-chain-contracts/contracts/GatewayTransferNative.sol#L530C14-L530C35) function when bridging native asset. Then, internal function `_handleFeeTransfer` is called: 
```solidity
   function withdrawToNativeChain(address zrc20, uint256 amount, bytes calldata message) external payable {
        if (zrc20 != _ETH_ADDRESS_) {
            require(
                IZRC20(zrc20).transferFrom(msg.sender, address(this), amount),
                "INSUFFICIENT ALLOWANCE: TRANSFER FROM FAILED"
            );
        }

       // ...

        // Transfer platform fees
        uint256 platformFeesForTx = _handleFeeTransfer(zrc20, amount); // platformFee = 5 <> 0.5%
        amount -= platformFeesForTx;
```
There, transfer helper is used to collect fees:
```solidity
    function _handleFeeTransfer(address zrc20, uint256 amount) internal returns (uint256 platformFeesForTx) {
        platformFeesForTx = (amount * feePercent) / 1000; // platformFee = 5 <> 0.5%
        TransferHelper.safeTransfer(zrc20, EddyTreasurySafe, platformFeesForTx);
    }
```
That's where the problem lies - since asset is native asset and not erc20 token, direct ETH transfer methods should be used instead of erc20-transfer:
```solidity
    function safeTransfer(address token, address to, uint value) internal {
        // bytes4(keccak256(bytes('transfer(address,uint256)')));
        (bool success, bytes memory data) = token.call(abi.encodeWithSelector(0xa9059cbb, to, value));
        require(success && (data.length == 0 || abi.decode(data, (bool))), 'TransferHelper: TRANSFER_FAILED');
    }
```
What happens instead is that low level call 'transfer(address,uint256)' is executed on `address(0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE)`. Since there's no code at the address, call will be successful even though no funds were moved. TX execution normally proceeds while protocol collected 0 fees.


### Internal Pre-conditions

None

### External Pre-conditions

User initiates a native asset bridge transaction by calling `withdrawToNativeChain` with `zrc20 = _ETH_ADDRESS_`

### Attack Path

1. User calls `withdrawToNativeChain(_ETH_ADDRESS_, amount, message)` with 0 value
2. Function skips token collection validation for native ETH
3. Platform calculates fee: `platformFeesForTx = (amount * feePercent) / 1000`
4. `_handleFeeTransfer` attempts to transfer fees but calls non-contract address `_ETH_ADDRESS_`
5. Low-level call returns success without actual transfer
6. Bridge proceeds with zero fees collected
Result: User bridges native asset, but protocol receives no revenue

### Impact

Medium severity - revenue loss for the protocol

### PoC

_No response_

### Mitigation

Implement proper native asset fee collection

## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/Skyewwww/omni-chain-contracts/pull/28

---
### Example 2

**Auto Label:** Failure to validate ownership or permissions in token transfers and vault access leads to unauthorized fund movement, collateral loss, or self-inflation, enabling attackers to exploit or drain protocol value.  

**Original Text Preview:**

## Security Analysis Report

## Severity
**Low Risk**

## Context
(No context files were provided by the reviewers)

## Description
In `Provisioner.sol#L342-L363`, the contract allows a vault manager to configure a set of allowed tokens for deposit. There is currently no restriction that prevents the vault’s own unit token (i.e., the ERC20 representing shares in the vault) from being listed as an allowed token.

If the vault's own token is mistakenly added to this list:
- Users or privileged actors could deposit vault units back into the vault itself.
- This could result in circular deposits, inflated vault unit balances, or broken accounting.
- It would violate expected asset-token separation and could lead to downstream insolvency or redemption failure.

This pattern resembles a recursive self-deposit and creates a risk of artificial vault growth disconnected from real asset inflows.

## Recommendation
Consider enforcing a check in the `setTokenConfig` function to prevent the vault's own asset (i.e., `address(this)` or `vault.asset()` if exposed) from being listed as an allowed deposit token. This would ensure that vault units cannot be re-deposited as if they were external assets. 

Combining this with the previously discussed `msg.sender != vault` check would close the loop on self-deposit and recursive inflation scenarios.

## Aera
Fixed in PR 333.

## Spearbit
Fix verified.

---
### Example 3

**Auto Label:** Failure to validate ownership or permissions in token transfers and vault access leads to unauthorized fund movement, collateral loss, or self-inflation, enabling attackers to exploit or drain protocol value.  

**Original Text Preview:**

## Security Report

## Severity 
**Low Risk**

## Context 
`SingleDepositorVault.sol#L27-L38`

## Description 
The `deposit()` function of `SingleDepositorVault` allows the owner or an authorized party to deposit funds into the vault. Before calling `deposit()`, the vault should have enough allowance to transfer the tokens from the depositor.

In the case where the depositor unintentionally approves the vault for more than needed, i.e., there is an excessive allowance after the call, they are exposed to the risk of a compromised guardian transferring their token to another address. Note that the compromised guardian still needs permission to call `transferFrom()`, etc., to exploit the excessive allowance.

## Recommendation 
Consider checking `tokenAmount.token.allowance(msg.sender, address(this)) == 0` at the end of each `safeTransferFrom()` call as a general way to prevent this issue regardless of whether the guardian is granted with a `transferFrom()` or similar permission.

## Aera 
Fixed in PR 288.

## Spearbit 
Verified.

---
