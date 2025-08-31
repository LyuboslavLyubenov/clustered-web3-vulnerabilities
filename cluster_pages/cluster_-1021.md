# Cluster -1021

**Rank:** #448  
**Count:** 7  

## Label
Failure to validate pending state or identities before allowing updates enables unauthorized actions, leading to unjustified settlements, rule mismatches, or orphaned commitments and resulting in denial of service or asset loss.

## Cluster Information
- **Total Findings:** 7

## Examples

### Example 1

**Auto Label:** Failure to validate pending state or identities before allowing updates enables unauthorized actions, leading to unjustified settlements, rule mismatches, or orphaned commitments and resulting in denial of service or asset loss.  

**Original Text Preview:**

## Summary

The current implementation of `UpdateWeightRunner` introduces a critical vulnerability in the protocol. If the `quantammAdmin` modifies the `UpdateWeightRunner`, it could lead to unexpected behavior where the protocol breaks. Specifically:

1. A new `UpdateWeightRunner` might have a different `quantammAdmin`, which would not align with the existing `Pool`.
2. The rule required by the new `UpdateWeightRunner` is not set because the rule is defined during the `Pool` initialization phase.

This issue creates inconsistencies in the protocol, potentially leading to a denial of service (DoS) for affected pools.

## Vulnerability Details

The vulnerability arises when the UpdateWeightRunner is changed, causing critical issues:

1. Admin Ownership Mismatch: The new UpdateWeightRunner may have a different quantAdmin, leading to conflicting authority and governance inconsistencies.

2. Missing Rules: The pool’s rules, set during initialization, are not carried over to the new UpdateWeightRunner. This prevents updates, effectively causing a denial-of-service (DoS) for the pool.

### Proof of Concept (POC)

Add the following test to `QuantAMMWeightedPool2TokenTest` to simulate the issue:

#### Initialization of the New `UpdateWeightRunner`:

```solidity
updateWeightRunner1 = new MockUpdateWeightRunner(owner, addr2, false); // Add this to the constructor
```

#### POC Test Case:

```solidity
MockUpdateWeightRunner updateWeightRunner1;

function testQuantAMMWeightedPoolGetNormalizedWeightsInitial_andThenChangeUpdateWeightRunner() public {
    QuantAMMWeightedPoolFactory.NewPoolParams memory params = _createPoolParams();
    params._initialWeights[0] = 0.6e18;
    params._initialWeights[1] = 0.4e18;

    (address quantAMMWeightedPool, ) = quantAMMWeightedPoolFactory.create(params);

    uint256[] memory weights = QuantAMMWeightedPool(quantAMMWeightedPool).getNormalizedWeights();

    int256;
    newWeights[0] = 0.6e18;
    newWeights[1] = 0.4e18;
    newWeights[2] = 0e18;
    newWeights[3] = 0e18;

    uint64;
    lambdas[0] = 0.2e18;

    int256;
    parameters0] = 0.2e18;

    address[][] memory oracles oracles[0][0]acle);

    MockMomentumRule momentumRule = new MockMomentumRule(owner);

    // Change UpdateWeightRunner
    vm.prank(owner);
    QuantAMMWeightedPool(quantAMMWeightedPool).setUpdateWeightRunnerAddress(address(updateWeightRunner1));
    
    QuantAMMWeightedPool(quantAMMWeightedPool).initialize(
        newWeights,
        IQuantAMMWeightedPool.PoolSettings(
            new IERC20 ,
            IUpdateRule(momentumRule),        oracles,
            60,
            lambdas,
            0.2e18,
            0.2e18,
            0.2e18,
            parameters,
            address(0)
        ),
        newWeights,
        newWeights,
        10
    );

    // Perform an update with the new runner
    vm.prank(owner);
    updateWeightRunner1.setApprovedActionsForPool(quantAMMWeightedPool, 1);
    updateWeightRunner1.performUpdate(quantAMMWeightedPool);
}
```

## Impact

Changing the `UpdateWeightRunner` leads to the following issues:

1. **Denial of Service (DoS):**\
   The new `UpdateWeightRunner` does not inherit the rule for the existing pool, rendering it non-functional.

2. **Unauthorized Updates:**\
   The `quantAdmin` of the initial `UpdateWeightRunner` can update the pool with the new `UpdateWeightRunner`, creating further inconsistencies.

These flaws disrupt the protocol and can lead to operational outages or malicious misuse.

## Tools Used

Manual review

## Recommendations

To address this vulnerability, update the `setUpdateWeightRunnerAddress` function to synchronize `quantammAdmin` and ensure the rule is correctly set during the update. Modify the function as follows:

### Updated Code

```diff
function setUpdateWeightRunnerAddress(address _updateWeightRunner) external override {
    require(msg.sender == quantammAdmin, "ONLYADMIN");
    updateWeightRunner = UpdateWeightRunner(_updateWeightRunner);
+   quantammAdmin = updateWeightRunner.quantammAdmin();
+   _setRule(); // Call set rule with the correct parameters
    emit UpdateWeightRunnerAddressUpdated(address(updateWeightRunner), _updateWeightRunner);
}
```

---
### Example 2

**Auto Label:** Failure to validate pending state or identities before allowing updates enables unauthorized actions, leading to unjustified settlements, rule mismatches, or orphaned commitments and resulting in denial of service or asset loss.  

**Original Text Preview:**

The `SofamonGachaMachine` contract includes a two-step process for minting wearable NFTs, involving a commit phase and a subsequent reveal phase. During the commit phase, a randomness request is made using an rng (random number generator) contract, and during the reveal phase, the random value associated with the previous request is retrieved from the rng contract.

The zSofamonGachaMachinez contract has a `setRNG` function that allows the contract owner to change the rng contract. However, if this function is called while there are pending commitments (i.e., commits that have not yet been revealed), those commitments will be unable to complete the reveal phase, effectively causing a Denial of Service (DoS) for honest users who have made those commitments.

Before allowing the rng contract to be changed, check if there are any pending commitments that have not yet been revealed. If there are pending commits, the `setRNG` function should revert the transaction and prevent the rng contract from being changed until all pending commits are revealed.

---
### Example 3

**Auto Label:** Failure to validate pending state or identities before allowing updates enables unauthorized actions, leading to unjustified settlements, rule mismatches, or orphaned commitments and resulting in denial of service or asset loss.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-09-symmio-v0-8-4-update-contest-judging/issues/40 

The protocol has acknowledged this issue.

## Found by 
xiaoming90
### Summary

The `settleUpnl` function does not include `notPartyB` modifier. As a result, unauthorized PartyB could settle PNL of other PartyBs and users in the system, causing disruption and breaking core protocol functionality. Unauthorized PartyB can prematurely settle PartyB's (victim) positions prematurely at times that are disadvantageous to PartyB, resulting in asset loss for them.

For example, in a highly volatile market, if PartyB’s positions temporarily incur a loss due to sudden market fluctuations, the Unauthorized PartyB could immediately settle these positions before the market has a chance to recover. This premature settlement forces PartyB to realize losses that might have otherwise been avoided if the positions had remained open.

### Root Cause

- The `settleUpnl` function does not include `notPartyB` modifier.

### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

The `settleUpnl` function can only be accessed by PartyB, as per the comment below. However, the function is not guarded by the [`notPartyB`](https://github.com/sherlock-audit/2024-09-symmio-v0-8-4-update-contest/blob/main/protocol-core/contracts/utils/Accessibility.sol#L18) modifier.

https://github.com/sherlock-audit/2024-09-symmio-v0-8-4-update-contest/blob/main/protocol-core/contracts/facets/Settlement/SettlementFacet.sol#L26

```solidity
File: SettlementFacet.sol
16: 	/**
17: 	 * @notice Allows Party B to settle the upnl of party A position for the specified quotes.
18: 	 * @param settlementSig The data struct contains quoteIds and upnl of parties and market prices
19: 	 * @param updatedPrices New prices to be set as openedPrice for the specified quotes.
20: 	 * @param partyA Address of party A
21: 	 */
22: 	function settleUpnl(
23: 		SettlementSig memory settlementSig,
24: 		uint256[] memory updatedPrices,
25: 		address partyA
26: 	) external whenNotPartyBActionsPaused notLiquidatedPartyA(partyA) {
27: 		uint256[] memory newPartyBsAllocatedBalances = SettlementFacetImpl.settleUpnl(settlementSig, updatedPrices, partyA);
..SNIP..
35: 	}
```

Instead, it depends on the `quoteLayout.partyBOpenPositions[msg.sender][partyA].length > 0` at Line 31 below, which is not a reliable method to determine whether the caller is a valid PartyB. The reason is that it is possible that a PartyB (e.g., one that might be removed due to malicious activities) that has already been removed from the system still has residual open positions. In this case, the position's length check will pass, and the unauthorized PartyB could continue to settle the PNL of other PartyBs and users in the system.

https://github.com/sherlock-audit/2024-09-symmio-v0-8-4-update-contest/blob/main/protocol-core/contracts/libraries/LibSettlement.sol#L24

```solidity
File: LibSettlement.sol
15: 	function settleUpnl(
..SNIP..
30: 		require(
31: 			isForceClose || quoteLayout.partyBOpenPositions[msg.sender][partyA].length > 0,
32: 			"LibSettlement: Sender should have a position with partyA"
33: 		);
```

### Impact

Unauthorized PartyB could settle PNL of other PartyBs and users in the system, causing disruption and breaking core protocol functionality. Unauthorized PartyB can prematurely settle PartyB's (victim) positions prematurely at times that are disadvantageous to PartyB, resulting in asset loss for them.

For example, in a highly volatile market, if PartyB’s positions temporarily incur a loss due to sudden market fluctuations, the Unauthorized PartyB could immediately settle these positions before the market has a chance to recover. This premature settlement forces PartyB to realize losses that might have otherwise been avoided if the positions had remained open.

### PoC

_No response_

### Mitigation

Include the `notPartyB` modifier to the `settleUpnl` function.



## Discussion

**MoonKnightDev**

the check in the libSettlement.sol file prevents the scenario mentioned: https://github.com/SYMM-IO/protocol-core/blob/eac73bf1d97df96bcd5b19bcc972792ef96c70e1/contracts/libraries/LibSettlement.sol#L30

---
