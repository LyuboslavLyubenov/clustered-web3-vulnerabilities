# Cluster -1178

**Rank:** #389  
**Count:** 13  

## Label
Missing access control and authorization checks enable unauthorized users or entities to manipulate core financial logic, leading to fund drains, unlimited leverage, or invalid asset approvals.

## Cluster Information
- **Total Findings:** 13

## Examples

### Example 1

**Auto Label:** Insufficient authorization validation allows unauthorized parties to perform critical actions, bypassing consent or operator approvals and enabling misuse of collateral or debt management.  

**Original Text Preview:**

<https://github.com/code-423n4/2024-03-taiko/blob/f58384f44dbf4c6535264a472322322705133b11/packages/protocol/contracts/L1/provers/GuardianProver.sol#L46> 

<https://github.com/code-423n4/2024-03-taiko/blob/f58384f44dbf4c6535264a472322322705133b11/packages/protocol/contracts/L1/libs/LibProving.sol#L192>

### Vulnerability details

The `GuardianProver` contract is a multisig that might contest any proof in some exceptional cases (bugs in the prover or verifier). To contest a proof, a predefined number of guardians should approve a hash of the message that includes `_meta` and `_tran`.

```solidity
function approve(
    TaikoData.BlockMetadata calldata _meta,
    TaikoData.Transition calldata _tran,
    TaikoData.TierProof calldata _proof
)
    external
    whenNotPaused
    nonReentrant
    returns (bool approved_)
{
    if (_proof.tier != LibTiers.TIER_GUARDIAN) revert INVALID_PROOF();

    bytes32 hash = keccak256(abi.encode(_meta, _tran));
    approved_ = approve(_meta.id, hash);

    if (approved_) {
        deleteApproval(hash);
        ITaikoL1(resolve("taiko", false)).proveBlock(_meta.id, abi.encode(_meta, _tran, _proof));
    }

    emit GuardianApproval(msg.sender, _meta.id, _tran.blockHash, approved_);
}
```

The issue arises from the fact that the approved message doesn't include the `_proof`. It means that the last approving guardian can provide any desired value in the `_proof`. The data from the `_proof` is used to determine whether it is necessary to return the liveness bond to the assigned prover or not:

```solidity
if (isTopTier) {
    // A special return value from the top tier prover can signal this
    // contract to return all liveness bond.
    bool returnLivenessBond = blk.livenessBond > 0 && _proof.data.length == 32
        && bytes32(_proof.data) == RETURN_LIVENESS_BOND;

    if (returnLivenessBond) {
        tko.transfer(blk.assignedProver, blk.livenessBond);
        blk.livenessBond = 0;
    }
}
```

As a result, the last guardian can solely decide whether to return the liveness bond to the assigned prover or not.

### Impact

The decision to return the liveness bond depends solely on the last guardian.

### Recommended Mitigation Steps

Consider including the `_proof` in the approved message.

```solidity
bytes32 hash = keccak256(abi.encode(_meta, _tran, _proof));
```

**[dantaik (Taiko) confirmed and commented](https://github.com/code-423n4/2024-03-taiko-findings/issues/205#issuecomment-2079069856):**
 >Now guardian provers must also agree on whether liveness bond should be returned bytes32(_proof.data) == LibStrings.H_RETURN_LIVENESS_BOND, so the hash they sign is now:
 >
 >bytes32 hash = keccak256(abi.encode(_meta, _tran, _proof.data));
 >
 >rather than the previous code:
 >
 >bytes32 hash = keccak256(abi.encode(_meta, _tran));
***

---
### Example 2

**Auto Label:** Missing access control and authorization checks enable unauthorized users or entities to manipulate core financial logic, leading to fund drains, unlimited leverage, or invalid asset approvals.  

**Original Text Preview:**

##### Description
Currently, any user can call the sweep function and get all tokens from the contract.
https://github.com/cryptoalgebra/Algebra/blob/bddd6487c86e0d6afef39638159dc403a91ba433/src/periphery/contracts/base/PeripheryPayments.sol#L32-L48
##### Recommendation
We recommend adding an admin role for calling this function.

---
### Example 3

**Auto Label:** Insufficient authorization validation allows unauthorized parties to perform critical actions, bypassing consent or operator approvals and enabling misuse of collateral or debt management.  

**Original Text Preview:**

## Security Vulnerability Report

## Severity
**High Risk**

## Context
`VaultImplementation.sol#L225`

## Description
In the `_validateCommitment()` function, the initial checks are intended to ensure that the caller who is requesting the lien is someone who should have access to the collateral that it's being taken out against. The caller also inputs a receiver, who will be receiving the lien. 

In this validation, this receiver is checked against the collateral holder, and the validation is approved in the case that `receiver == holder`. However, this does not imply that the collateral holder wants to take this loan.

This opens the door to a malicious lender pushing unwanted loans on holders of collateral by calling `commitToLien` with their `collateralId`, as well as their address set to the receiver. This will pass the `receiver == holder` check and execute the loan.

In the best case, the borrower discovers this and quickly repays the loan, incurring a fee and a small amount of interest. In the worst case, the borrower doesn't know this happens, and their collateral is liquidated.

## Recommendation
Only allow calls from the holder or operator to lead to valid commitments:

```solidity
address holder = CT.ownerOf(collateralId);
address operator = CT.getApproved(collateralId);

if (
    msg.sender != holder &&
    receiver != holder &&
    receiver != operator &&
    !ROUTER().isValidVault(receiver)
) {
    msg.sender != operator &&
    CT.isApprovedForAll(holder, msg.sender)
) {
    if (operator != address(0)) {
        require(operator == receiver);
    } else {
        require(CT.isApprovedForAll(holder, receiver));
    }
} else {
    revert NotApprovedForBorrow();
}
```

---
