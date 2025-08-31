# Cluster -1119

**Rank:** #81  
**Count:** 211  

## Label
Failure to validate or control cross-chain inputs and message execution leads to unauthorized operations, inconsistent address generation, and loss of state integrity, enabling reentrancy, privilege escalation, and unintended fund transfers.

## Cluster Information
- **Total Findings:** 211

## Examples

### Example 1

**Auto Label:** Failure to validate or control cross-chain inputs and message execution leads to unauthorized operations, inconsistent address generation, and loss of state integrity, enabling reentrancy, privilege escalation, and unintended fund transfers.  

**Original Text Preview:**

Setting `interchainSecurityModule` to `address(0)` causes the protocol to use Hyperlane's default ISM, which can be changed by the Hyperlane team at any time. This creates a security risk without the protocol's control.

```solidity
    function setInterchainSecurityModule(IInterchainSecurityModule _interchainSecurityModule) external requiresAuth {
        interchainSecurityModule = _interchainSecurityModule;
        emit SetInterChainSecurityModule(address(_interchainSecurityModule));
    }
```

```solidity
    function setDefaultIsm(address _module) public onlyOwner {
        require(
            Address.isContract(_module),
            "Mailbox: default ISM not contract"
        );
        defaultIsm = IInterchainSecurityModule(_module);
        emit DefaultIsmSet(_module);
    }
```

For example, if Hyperlane's team sets the default ISM to `NoopIsm`, any attacker could call `mailbox.process()` to mint unauthorized shares since NoopIsm performs no security verification.

NoopIsm: [link](https://github.com/hyperlane-xyz/hyperlane-monorepo/blob/main/solidity/contracts/isms/NoopIsm.sol)

The likelihood is very low, however the protocol should monitor the use ISM or set a secure ISM itself.

---
### Example 2

**Auto Label:** Unauthorized access and validation bypasses in cross-chain message handling lead to unauthorized operations, fund misuse, and transaction loss due to insufficient sender, caller, and message state checks.  

**Original Text Preview:**

Inside `MultiChainTellerBase`, there are several operations to allow or stop receiving messages from a configured chain (`allowMessagesFrom` / `allowMessagesTo`). However, it does not verify if `targetTeller` is a non-empty address when `allowMessagesFrom` / `allowMessagesTo` is set to true. This could cause issues, as `mailbox.dispatch` will set `msgRecipient` to an empty address if misconfigured. Consider adding additional verification.

---
### Example 3

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
