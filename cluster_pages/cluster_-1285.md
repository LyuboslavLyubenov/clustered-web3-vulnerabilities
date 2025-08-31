# Cluster -1285

**Rank:** #442  
**Count:** 8  

## Label
Failure to validate critical state invariants during parameter updates, enabling invalid configurations, inconsistent pricing, and potential data leakage or system instability.

## Cluster Information
- **Total Findings:** 8

## Examples

### Example 1

**Auto Label:** Failure to validate critical state invariants during parameter updates, enabling invalid configurations, inconsistent pricing, and potential data leakage or system instability.  

**Original Text Preview:**

### Summary

The `AdminFacet` contract contains an inconsistency in how it enforces permanent rollup state constraints across different fee-related functions. While `setPubdataPricingMode()` and `makePermanentRollup()` properly enforce that permanent rollups must use the `Rollup` pricing mode, the `changeFeeParams()` function lacks this validation. This creates a potential state invariant violation where a permanent rollup chain could have an invalid pricing mode.

While the current implementation of `changeFeeParams()` prevents direct changes to the `pubdataPricingMode`, it doesn't validate that the existing/incoming mode is correct for permanent rollups. This means that if a chain becomes a permanent rollup through state migration (via `forwardedBridgeMint()`), there's no validation ensuring the fee parameters maintain the required `Rollup` pricing mode.

This inconsistency could lead to:

1. State invariant violations where a permanent rollup operates with an invalid pricing mode
2. Potential security issues if other parts of the system assume permanent rollups always use `Rollup` pricing mode
3. Difficulty in upgrading or maintaining the system due to inconsistent state validation

## Proof of Concept

<https://github.com/Cyfrin/2024-10-zksync/blob/cfc1251de29379a9548eeff1eea3c78267288356/era-contracts/l1-contracts/contracts/state-transition/chain-deps/facets/Admin.sol#L100>

The issue can be demonstrated through the following sequence of operations:

```solidity
// Initial state: Chain is not a permanent rollup, using Validium mode
FeeParams memory initialParams = FeeParams({
    pubdataPricingMode: PubdataPricingMode.Validium,
    ...
});
s.feeParams = initialParams;
s.isPermanentRollup = false;

// 1. Chain becomes permanent rollup through migration
ZKChainCommitment memory commitment = ZKChainCommitment({
    isPermanentRollup: true,
    ...
});
adminFacet.forwardedBridgeMint(abi.encode(commitment), true);
// Now s.isPermanentRollup = true, but fee params still use Validium mode

// 2. Update fee parameters
FeeParams memory newParams = FeeParams({
    pubdataPricingMode: PubdataPricingMode.Validium, // Same as old mode
    ...
});
// This call succeeds despite invalid state
adminFacet.changeFeeParams(newParams);
```

Relevant code references showing the inconsistency:

1. `setPubdataPricingMode()` enforces the constraint:

```solidity
if (s.isPermanentRollup && _pricingMode != PubdataPricingMode.Rollup) {
    revert IncorrectPricingMode();
}
```

1. `changeFeeParams()` lacks the same validation:

```solidity
if (_newFeeParams.pubdataPricingMode != oldFeeParams.pubdataPricingMode) {
    revert InvalidPubdataPricingMode();
}
```

## Recommended mitigation steps

Add permanent rollup state validation to the `changeFeeParams()` function:

```solidity
function changeFeeParams(FeeParams calldata _newFeeParams) external onlyAdminOrChainTypeManager onlyL1 {
    if (_newFeeParams.maxPubdataPerBatch < _newFeeParams.priorityTxMaxPubdata) {
        revert PriorityTxPubdataExceedsMaxPubDataPerBatch();
    }

    FeeParams memory oldFeeParams = s.feeParams;

    // we cannot change pubdata pricing mode
    if (_newFeeParams.pubdataPricingMode != oldFeeParams.pubdataPricingMode) {
        revert InvalidPubdataPricingMode();
    }

    // Add validation for permanent rollup state
    if (s.isPermanentRollup && _newFeeParams.pubdataPricingMode != PubdataPricingMode.Rollup) {
        revert IncorrectPricingMode();
    }

    s.feeParams = _newFeeParams;

    emit NewFeeParams(oldFeeParams, _newFeeParams);
}
```

---
### Example 2

**Auto Label:** Misconfiguration due to unbounded parameters or flawed constraints, leading to invalid state transitions, reduced incentives, and erosion of protocol trust and functionality.  

**Original Text Preview:**

## InfraredVault.sol - Issue Report

## Context
InfraredVault.sol#L25

## Description
The `MAX_NUM_REWARD_TOKENS` constant is set to 10 in the InfraredVault contract, which contradicts the Berachain documentation that states there is a limit of 3 incentive assets per gauge (Documentation). This discrepancy can lead to potential issues with compliance and functionality when more than 3 reward tokens are added.

## Recommendation
Consider updating the `MAX_NUM_REWARD_TOKENS` constant to 3 to align with the documentation. Additionally, review the contract and associated documentation to ensure consistency in the limits imposed on the number of incentive assets. This change will ensure the contract remains compliant with the stated criteria for incentives and prevents potential issues from exceeding the documented limits.

---
### Example 3

**Auto Label:** Misconfiguration due to unbounded parameters or flawed constraints, leading to invalid state transitions, reduced incentives, and erosion of protocol trust and functionality.  

**Original Text Preview:**

## Context: Infrared.sol#L197-L201

## Description
The comment on line 197 in the initialize function of the Infrared contract incorrectly states that the IBGT vault can have IBGT and IRED rewards. In reality, the IBGT vault can also have Honey (Bera native stablecoin) as a reward token. This incorrect comment can lead to misunderstandings about the functionality and supported reward tokens of the IBGT vault.

## Recommendation
Update the comment to accurately reflect that the IBGT vault can have IBGT, IRED, and Honey as reward tokens. Ensure that all comments within the contract correctly describe the associated code behavior to maintain clarity and prevent misunderstandings.

---
