# Cluster -1355

**Rank:** #275  
**Count:** 30  

## Label
Lack of access control and input validation enabling unauthorized modifications or bypasses of critical security checks, leading to fund exposure, protocol manipulation, or unauthorized operations.

## Cluster Information
- **Total Findings:** 30

## Examples

### Example 1

**Auto Label:** Unauthorized function execution via insufficient access control and signature validation, enabling attackers to exploit or drain funds through arbitrary or impersonated calls.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** High

## Description

The `OmoVault.sol` contract implements some principles of the ERC7540 standard, Asynchronous redeem shares for assets.
However, users are still able to withdraw directly using `ERC4626.sol#withdraw()`

Also, `_validateSignature()` function in `DynamicAccount.sol` contract accepts the user wallet as a signer, through the Entry Point.
So, the user has the same privileges as the agent.

Malicious users can call `withdraw()` from the vault directly to receive their funds.
Next, he will transfer all the funds and the UniswapV3 positions out of their Dynamic Account

As a result, he will duple his first deposit in the vault.

## Recommendations

1- Override `withdraw()` function in `OmoVault.sol` and make it revert.
2- In `_validateSignature()` you need to check what function can be called if the signer is the owner

---
### Example 2

**Auto Label:** Failure to validate state or funds before gas consumption enables attackers to exploit transactions, leading to fund loss, denial of service, or unauthorized state changes.  

**Original Text Preview:**

## Severity: Critical Risk

## Context
`core/vm/evm.go#L813-L862`

## Description
Due to the location of the `CanTransfer()` check in `(evm *EVM) Restore()`, it is possible for a malicious entity to grief honest restoration servers for a significant amount of gas. This can become a more serious issue if a large portion of restoration servers run out of gas and their accounts expire before owners realize. A competing restoration server may want to do this to harm its competition.

An example attack could be a malicious actor creating a large number of accounts (100+) and seeding them all with 10 OVER. It would then need to let some time pass and the accounts go dormant (the longer the accounts are dormant, the larger the restoration proof and the more effective this attack will be). The entity then could create its own restoration proofs with the fees costing the entire 10 OVER for each account. This would effectively collect all of the account balances into its own account. It could wait until it either controls a validator that is the proposer for the next block, or it can use increased priority fees to make sure that its transactions are placed early in the next block. It can send all of its own `restorationTX`'s at the same time that it makes restoration requests to all of the other known restoration servers in the network.

The remaining honest restoration servers will query their local nodes that will not have their state updated yet with the malicious entity's self-created `restorationTX`'s. All checks will pass including `checkFeePayable()` which will see 10 OVER in each account. The honest restoration servers will all create and submit `restorationTX`'s for the 100+ accounts. By the time they are processed by the EVM, the malicious entity's `restorationTX`'s will have already processed and the victim restoration servers' `restorationTX`'s will fail at the requester’s balance check in `(evm *EVM) Restore()`. When this happens, all of the honest restoration servers' gas will be consumed:

```go
// The sender of the restore data has to have enough balance to send the restoration fee
if restoreData.FeeRecipient != nil && restoreData.Fee.Sign() != 0 {
    if !evm.Context.CanTransfer(evm.StateDB, sender, restoreData.Fee) {
        err = ErrInsufficientBalance
    } else {
        evm.Context.Transfer(evm.StateDB, sender, *restoreData.FeeRecipient, restoreData.Fee)
    }
}
if err != nil {
    evm.StateDB.RevertToSnapshot(snapshot)
    if err != ErrExecutionReverted {
        gas = 0
    }
}
```

The result of this attack is that the restoration servers will have their gas funds burned without being able to recoup the fees required to cover their loss. In its current form, the max gas that can be grieved is only limited by the base fee and the size of the proof (length of time the account has been dormant). Using hundreds or thousands of accounts requiring large proofs can enable an attacker to trick honest restoration servers to burn all of their balances in gas.

## Recommendation
Move the following check up to before the gas consumption portion of `(evm *EVM) Restore()`:

```go
if restoreData.SourceEpoch != epochCoverage {
    return nil, gas, ErrInvalidSourceEpoch
}
```

## Overprotocol
Fixed in commit `b4625c76`.

## Spearbit
The Spearbit Review team confirmed that this issue has been mitigated by moving the `CanTransfer()` and `restoreData.SourceEpoch != epochCoverage` checks up to before the gas proof is consumed in `Restore()`. These changes were introduced in commit `b4625c76`.

---
### Example 3

**Auto Label:** Failure to validate state or funds before gas consumption enables attackers to exploit transactions, leading to fund loss, denial of service, or unauthorized state changes.  

**Original Text Preview:**

## Goldsand.sol Analysis

## Context
Goldsand.sol#L164-L174

## Description
In `Goldsand.sol`, we have fallback and receive functions to handle any incoming ETH transfers by calling `fund`, applying the transferred amount to the `msg.sender`'s `funderToBalance`:

```solidity
/**
 * @notice Fallback function
 * @dev Handles Ether sent directly to the contract.
 */
fallback() external payable whenNotPaused {
    fund();
}

/**
 * @notice Receive function
 * @dev Handles Ether sent directly to the contract.
 */
receive() external payable whenNotPaused {
    fund();
}

function fund() public payable whenNotPaused {
    if (msg.value < minEthDeposit) {
        revert TooSmallDeposit();
    }
    funderToBalance[msg.sender] += msg.value;
    emit Funded(msg.sender, msg.value);
}
```

ETH is occasionally transferred from the `WithdrawalVault` to `Goldsand` via `withdrawETHToGoldsand`, which works via a low-level call that doesn't include a function selector. This call gets routed to the fallback or receive method where `fund` is called. This is unexpected as transfers from `WithdrawalVault` are not intended to be used for funding.

As a result, we increment `funderToBalance[withdrawalVaultAddress]` by the amount transferred. Since we can only decrement `funderToBalance` values via `operatorWithdrawETHForUser`, and that only decrements the amount for one account, we either can't clear the amount of the actual user to withdraw for or `funderToBalance[withdrawalVaultAddress]`.

## Recommendation
Include an exception in fallback and receive which doesn’t call `fund` if the `msg.sender` is `withdrawalVaultAddress`:

```solidity
/**
 * @notice Fallback function
 * @dev Handles Ether sent directly to the contract.
 */
fallback() external payable whenNotPaused {
    if (msg.sender == withdrawalVaultAddress) return;
    fund();
}

/**
 * @notice Receive function
 * @dev Handles Ether sent directly to the contract.
 */
receive() external payable whenNotPaused {
    if (msg.sender == withdrawalVaultAddress) return;
    fund();
}
```

## Inshallah
Fixed in commit `43543c93`.

## Cantina Managed
This has been fixed by no longer withdrawing from `WithdrawalVault` to `Goldsand`.

---
