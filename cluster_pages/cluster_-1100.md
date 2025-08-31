# Cluster -1100

**Rank:** #300  
**Count:** 24  

## Label
Missing validation of critical contract addresses leads to unauthorized state transitions, inconsistent operations, or loss of assets due to invalid or stale vaults or logic dependencies.

## Cluster Information
- **Total Findings:** 24

## Examples

### Example 1

**Auto Label:** Hardcoded or incorrect addresses in contracts lead to failed interactions, unintended transfers, or loss of funds due to misalignment with actual on-chain token or protocol addresses.  

**Original Text Preview:**

## \[L-01] Incorrect event emitted in \`setUpdateWeightRunnerAddress()\` function

## Summary

Incorrect event emitted in \`setUpdateWeightRunnerAddress()\` function. When changing the `UpdateWeightRunner.sol` address, the event should emit 2 params - old and new addresses:

```solidity
event UpdateWeightRunnerAddressUpdated(address indexed oldAddress, address indexed newAddress);
```

The issue is that this function does not store the old address in memory, but returns the updated `updateWeightRunner` variable as first param instead of old `updateWeightRunner`:

```solidity
function setUpdateWeightRunnerAddress(address _updateWeightRunner) external override {
        require(msg.sender == quantammAdmin, "ONLYADMIN");
        updateWeightRunner = UpdateWeightRunner(_updateWeightRunner);
        emit UpdateWeightRunnerAddressUpdated(address(updateWeightRunner), _updateWeightRunner); 
    }
```

## Impact

Frontend or other off-chain services may display incorrect values, potentially misleading users, disrupt update tracking and make it difficult to find updates.

## Recommendations

```diff
function setUpdateWeightRunnerAddress(address _updateWeightRunner) external override {
        require(msg.sender == quantammAdmin, "ONLYADMIN");
+       address oldUpdateWeightRunner = updateWeightRunner;
        updateWeightRunner = UpdateWeightRunner(_updateWeightRunner);
-       emit UpdateWeightRunnerAddressUpdated(address(updateWeightRunner), _updateWeightRunner);
+       emit UpdateWeightRunnerAddressUpdated(oldUpdateWeightRunner, _updateWeightRunner);  
    }
```

---
### Example 2

**Auto Label:** Misconfigured contract addresses lead to failed interactions, causing operational failures, lost fees, and compromised protocol functionality due to incorrect or missing address returns.  

**Original Text Preview:**

## Severity

**Impact:** Low

**Likelihood:** High

## Description

The `DSRStrategy` declares incorrect contract addresses for `TOKEN_ADDRESS` and `DSR_DEPOSIT_ADDRESS`, which are intended to represent the `sDAI` contract for depositing `DAI` tokens and earning yield.

```solidity
//mainnet
address public constant TOKEN_ADDRESS=0x3F1c547b21f65e10480dE3ad8E19fAAC46C95034;

address public constant DSR_DEPOSIT_ADDRESS=0x3F1c547b21f65e10480dE3ad8E19fAAC46C95034;
```

The `DSRStrategy` is primarily used as the implementation strategy for the DepositUSD contract on the Ethereum mainnet. However, the current addresses point to an [account](https://etherscan.io/address/0x3F1c547b21f65e10480dE3ad8E19fAAC46C95034) with no code.

This will prevent all operations and interactions with the intended address.

## Recommendations

Update the `TOKEN_ADDRESS` and `DSR_DEPOSIT_ADDRESS` to the correct address:
`sDAI: 0x83F20F44975D03b1b09e64809B757c47f942BEeA` (ETH Mainnet).

---
### Example 3

**Auto Label:** Missing validation of critical contract addresses leads to unauthorized state transitions, inconsistent operations, or loss of assets due to invalid or stale vaults or logic dependencies.  

**Original Text Preview:**

## Review of Contract Parameters

## Context
- **Files**: 
  - SimpleSwapFacility.sol (Lines 72-79)
  - Vault.sol (Lines 22-31)

## Description
The parameters `swapFacility` and `token` are explicit, lacking zero-address validation. The `token` is implicitly checked; if it is not a valid or at least masquerading ERC20 contract, the function will revert.

However, the `swapFacility` could be set to `address(0)`, which would leave the Vault in an invalid state. This situation would result in a loss of any collateral swapped via the associated SwapFacility contract until an upgrade is performed to correct it.

## Recommendation
- Explicitly implement zero-address validation for `swapFacility`.
- Consider utilizing OpenZeppelin's `isContract` check for `initialAuthority` or any other parameters expected to be pre-deployed contracts, which may also apply to `swapFacility`.
- Additionally, it would be good practice for the `swapFacility` to check that it has a sufficient threshold of transfer approval from the Vault it is initializing.

## Responses
- **Bitcorn**: Acknowledged. Will retain the existing behavior and leverage script-based confirmation of correct parameters prior to deployment.
- **Cantina Managed**: Acknowledged.

---
