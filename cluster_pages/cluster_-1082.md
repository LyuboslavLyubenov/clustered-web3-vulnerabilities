# Cluster -1082

**Rank:** #150  
**Count:** 93  

## Label
Failure to validate or properly configure delegate authority in cross-chain messaging enables unauthorized slippage control and transaction manipulation, leading to loss of operational control and potential fund freezing.

## Cluster Information
- **Total Findings:** 93

## Examples

### Example 1

**Auto Label:** Unauthorized access and validation bypasses in cross-chain message handling lead to unauthorized operations, fund misuse, and transaction loss due to insufficient sender, caller, and message state checks.  

**Original Text Preview:**

Inside `MultiChainTellerBase`, there are several operations to allow or stop receiving messages from a configured chain (`allowMessagesFrom` / `allowMessagesTo`). However, it does not verify if `targetTeller` is a non-empty address when `allowMessagesFrom` / `allowMessagesTo` is set to true. This could cause issues, as `mailbox.dispatch` will set `msgRecipient` to an empty address if misconfigured. Consider adding additional verification.

---
### Example 2

**Auto Label:** Lack of input validation and cryptographic integrity in cross-chain message handling enables tampering, unauthorized state changes, and divergent on-chain/off-chain behavior.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/745 

## Found by 
Hueber, Kirkeelee, Kvar, Phaethon, molaratai, moray5554, newspacexyz, xiaoming90, yoooo, zxriptor

### Summary

-

### Root Cause

-

### Internal Pre-conditions

-

### External Pre-conditions

-

### Attack Path

The protocol team will transfer ETH to the router to be used as gas for LayerZero's cross-chain messaging. All cross-chain operations (e.g., `borrowCrossChain`, `liquidateCrossChain`, `repayCrossChainBorrow`) in the `CrossCahinRouter` contract require ETH residing in the contract in order for the LayerZero's `_lzSend` cross-chain messaging to work because any cross-chain message send requires a fee.

However, the problem with this design is that a malicious user can drain all the ETH residing on the `CrossChainRouter` contract intended for LayerZero fee by making many unnecessary cross-chain operations, likely with a very small amount (e.g., opening a dust cross-chain borrow position or dust repayment)

When no ETH is residing in the router, all Cross-Chain operations on the router will be fail/revert and be DOSed.

https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L804

```solidity
File: CrossChainRouter.sol
801:     /**
802:      * @notice Sends LayerZero message
803:      */
804:     function _send(
805:         uint32 _dstEid,
806:         uint256 _amount,
807:         uint256 _borrowIndex,
808:         uint256 _collateral,
809:         address _sender,
810:         address _destlToken,
811:         address _liquidator,
812:         address _srcToken,
813:         ContractType ctype
814:     ) internal {
815:         bytes memory payload =
816:             abi.encode(_amount, _borrowIndex, _collateral, _sender, _destlToken, _liquidator, _srcToken, ctype);
817: 
818:         bytes memory options = OptionsBuilder.newOptions().addExecutorLzReceiveOption(1_000_000, 0);
819: 
820:         _lzSend(_dstEid, payload, options, MessagingFee(address(this).balance, 0), payable(address(this)));
821:     }
```

### Impact

Loss of assets and core feature broken. It can lead to the loss of assets because if liquidation can be blocked or delayed, it will result in a loss for the protocol. If repayment cannot be performed or delayed by malicious user, users cannot repay or cannoty repay their debt in a timely manner, causing their debt to accumulate and their account eventually get liquidated unfairly.

### PoC

_No response_

### Mitigation

_No response_

---
### Example 3

**Auto Label:** Unauthorized access and validation bypasses in cross-chain message handling lead to unauthorized operations, fund misuse, and transaction loss due to insufficient sender, caller, and message state checks.  

**Original Text Preview:**

The `ForwarderBase` contract has the `setCrossDomainAdmin` [function](https://github.com/across-protocol/contracts/blob/5a0c67c984d19a3bb843a4cec9bb081734583dd1/contracts/chain-adapters/ForwarderBase.sol#L69) which changes the `crossDomainAdmin` state [variable](https://github.com/across-protocol/contracts/blob/5a0c67c984d19a3bb843a4cec9bb081734583dd1/contracts/chain-adapters/ForwarderBase.sol#L20). This variable is used to give access control to almost all functions within the contract, including those that are meant to be used to complete L1->L3 message passing and asset transfers. When calling `setCrossDomainAdmin`, it is assumed that there are no outstanding operations that need to be passed to another chain. This is because when operations arrive at L2, the original sender of those is the old `crossDomainAdmin` and will be blocked from continuing further.

In order to raise awareness about such behavior, consider documenting this edge case in the `setCrossDomainAdmin` function.

***Update:** Resolved in [pull request #729](https://github.com/across-protocol/contracts/pull/729) at commit [4ba3439](https://github.com/across-protocol/contracts/pull/729/commits/4ba34394de24cf79cdce84d9ceccfe61e2b21d03).*

---
