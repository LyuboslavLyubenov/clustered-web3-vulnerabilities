# Cluster -1308

**Rank:** #237  
**Count:** 41  

## Label
Improper initialization and lack of validation during contract setup enable attackers to manipulate state, bypass access controls, or exploit uninitialized variables, leading to unauthorized fund access or state corruption.

## Cluster Information
- **Total Findings:** 41

## Examples

### Example 1

**Auto Label:** Failure to initialize state variables leads to undefined behavior, incorrect logic, or exploitable state inconsistencies during contract execution.  

**Original Text Preview:**

## IntentSource Contract Issue

## Context
**Location:** IntentSource.sol#L204-L206

## Summary
The IntentSource contract allows the same intent to be published multiple times due to the missing claim state initialization. This occurs because the default value for claim status (0) is the same as the Initiated status, making it impossible to distinguish between uninitialized and newly initiated intents.

## Description
In the IntentSource contract's `publishIntent()` function, the check to prevent duplicate intents relies on comparing the claim status against `ClaimStatus.Initiated`:

```solidity
function publishIntent(
    Intent calldata intent,
    bool fund
) external payable returns (bytes32 intentHash) {
    // ...
    (intentHash, routeHash, ) = getIntentHash(intent);
    if (claims[intentHash].status != uint8(ClaimStatus.Initiated)) {
        revert IntentAlreadyExists(intentHash);
    }
    emit IntentCreated(
        intentHash,
        route.salt,
        route.source,
        route.destination,
        route.inbox,
        route.tokens,
        route.calls,
        reward.creator,
        reward.prover,
        reward.deadline,
        reward.nativeValue,
        reward.tokens
    );
    // ...
}
```

An issue arises because default values in Solidity are initialized to 0, which is the same value as the Initiated status. This means `claims[intentHash].status` returns 0 for both uninitialized and initiated intents. As a result, intents can be published multiple times before being claimed or refunded, potentially leading to duplicate funding or causing confusion for fillers monitoring intent creation events.

## Impact Explanation
The impact is low. While this doesn’t directly lead to loss of funds, it can cause operational issues for fillers relying on intent creation events and lead to accidental duplicate funding if users don't track their own intent publications.

## Likelihood Explanation
The likelihood is medium. This issue manifests by default in the normal operation of the protocol. Any intent can be republished until it reaches a terminal state (Claimed or Refunded). This could happen accidentally through UI errors, transaction retries, or user confusion about the publishing status of their intents.

## Recommendation
Add an Uninitialized state (0) to the claim status enum and make Initiated a non-zero value. Then modify the `publishIntent()` function to properly set the initial state:

```solidity
enum ClaimStatus {
    Uninitialized,
    Initiated,
    Claimed,
    Refunded
}

function publishIntent(
    Intent calldata intent,
    bool fund
) external payable returns (bytes32 intentHash) {
    // ...
    (intentHash, routeHash, ) = getIntentHash(intent);
    if (claims[intentHash].status != uint8(ClaimStatus.Uninitialized)) {
        revert IntentAlreadyExists(intentHash);
    }
    claims[intentHash].status = uint8(ClaimStatus.Initiated);
    emit IntentCreated(
        intentHash,
        route.salt,
        route.source,
        route.destination,
        route.inbox,
        route.tokens,
        route.calls,
        reward.creator,
        reward.prover,
        reward.deadline,
        reward.nativeValue,
        reward.tokens
    );
    // ...
}
```

This change ensures that new intents start in an Uninitialized state, publishing explicitly sets them to Initiated, and duplicate publishing attempts will fail. The result is a more robust intent lifecycle tracking system that prevents unintended republishing of intents.

## Eco
We'll make the change on the `7683OriginSettler`, but outside of that, there is no case where somebody would double fund an intent. Creating a duplicate intent is not really that big of a problem (and since we're not storing state, it will be difficult to prevent it, since publishing an intent is not even a required step) and it won't be possible for a duplicate intent to be fulfilled, so we don't feel a need to change this. The state names are confusing though; we'll look into those.

## Cantina Managed
Acknowledged.

---
### Example 2

**Auto Label:** Improper initialization and lack of validation during contract setup enable attackers to manipulate state, bypass access controls, or exploit uninitialized variables, leading to unauthorized fund access or state corruption.  

**Original Text Preview:**

#### Resolution

The Linea team has fixed the finding in [PR 353](https://github.com/Consensys/linea-monorepo/pull/353/files).


#### Description

In the `initialize` function of the `LineaRollup` contract, the genesis `shnarf`(`GENESIS_SHNARF`) is hard-coded, it cannot work with new networks that has a different genesis `shnarf` without modifying the contract, thus restricts the contract’s adaptability and interoperability across different networks.

#### Examples

**contracts/contracts/LineaRollup.sol:L143-L143**

```
currentFinalizedShnarf = GENESIS_SHNARF;

```

**contracts/contracts/LineaRollup.sol:L398-L414**

```
function _computeShnarf(
  bytes32 _parentShnarf,
  bytes32 _snarkHash,
  bytes32 _finalStateRootHash,
  bytes32 _dataEvaluationPoint,
  bytes32 _dataEvaluationClaim
) internal pure returns (bytes32 shnarf) {
  assembly {
    let mPtr := mload(0x40)
    mstore(mPtr, _parentShnarf)
    mstore(add(mPtr, 0x20), _snarkHash)
    mstore(add(mPtr, 0x40), _finalStateRootHash)
    mstore(add(mPtr, 0x60), _dataEvaluationPoint)
    mstore(add(mPtr, 0x80), _dataEvaluationClaim)
    shnarf := keccak256(mPtr, 0xA0)
  }
}

```
#### Recommendation

Remove the hard-coded genesis `shnarf`, compute it from a parameter passed in the `initialize` function ( `_initializationData.initialStateRootHash`).

---
### Example 3

**Auto Label:** Redundant or incomplete initialization leading to state corruption, undefined behavior, or unauthorized control due to improper sequencing or missing logic in contract deployment.  

**Original Text Preview:**

## Code Review Summary

## Context
**File**: WrappedBitcornNativeOFTAdapter.sol  
**Line**: 61

## Description
The `initialize` function in `WrappedBitcornNativeOFTAdapter.sol` calls `__ERC20Pausable_init()` twice, which is unnecessary and indicates a code quality issue.

## Recommendation
Remove one of the duplicate calls:
```solidity
function initialize(address initialAuthority, address _delegate) public initializer {
    __Ownable_init(_delegate);
    __OFTCore_init(_delegate);
    __ERC20_init("Wrapped Bitcorn OFT", "WBTCN");
    __ERC20Pausable_init(); // Keep only one call
    __ERC20Permit_init("Wrapped Bitcorn OFT");
    _initializeAuthority(initialAuthority);
    __ReentrancyGuard_init_unchained();
}
```

## Bitcorn
Fixed in commit `83d0dcbc`.

## Cantina Managed
Verified fix.

---
