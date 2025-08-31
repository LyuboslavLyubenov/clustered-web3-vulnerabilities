# Cluster -1184

**Rank:** #325  
**Count:** 21  

## Label
Improper state mutation due to memory vs. storage misuse leads to out-of-bounds access, incorrect state updates, and overpayments, enabling financial loss and inconsistent ledger tracking.

## Cluster Information
- **Total Findings:** 21

## Examples

### Example 1

**Auto Label:** State inconsistency due to improper sequencing, validation, or immutability handling, leading to incorrect state exposure, transaction failures, or financial loss.  

**Original Text Preview:**

## Context
(No context files were provided by the reviewer)

## Description/Recommendation
- **IMech.sol#L15-L21:** `maxDeliveryRate` and `paymentType` can be defined as `view`. This will guarantee that when those interface endpoints are used, `staticcall` is made instead of a regular call.
- **IStaking.sol:** `IStaking.sol` does not seem to be used and perhaps can be removed.

## Valory
Fixed on PR 94.

## Cantina Managed
Fix verified.

---
### Example 2

**Auto Label:** State mutation flaws causing incorrect state persistence, inconsistent variable updates, and logic errors that lead to fund loss, improper access, or inaccurate calculations.  

**Original Text Preview:**

##### Description

The current implementation logic of the `setReservedName` function in the **ReservedRegistry** contract allows for duplicate entries in the list of reserved names. If the same name is mistakenly reserved multiple times, it results in redundant entries in the internal list `_reservedNamesList` and unnecessary increments of the counter `_reservedNamesCount`. This can result in inefficient gas usage and an inaccurate count of reserved names, potentially impacting the performance and operational cost of the contract.

  

Code Location
-------------

The `setReservedName` function does not check if the name has already been reserved before adding it to the list and incrementing the count:

```
function setReservedName(string calldata name_) public onlyOwner {
  bytes32 labelHash_ = keccak256(abi.encodePacked(name_));
  _reservedNames[labelHash_] = name_;
  _reservedNamesList.push(labelHash_);
  _reservedNamesCount++;
}
```

##### BVSS

[AO:A/AC:L/AX:M/R:N/S:U/C:N/A:N/I:L/D:L/Y:N (2.1)](/bvss?q=AO:A/AC:L/AX:M/R:N/S:U/C:N/A:N/I:L/D:L/Y:N)

##### Recommendation

Introduce a check in the `setReservedName` function to verify if the name has already been reserved before adding it to the list and incrementing the count. This would prevent the redundant updates to the state and improve gas efficiency.

##### Remediation

**SOLVED:** The **Beranames team** solved the issue in the specified commit id.

##### Remediation Hash

<https://github.com/Beranames/beranames-contracts-v2/pull/94/commits/32934129e1ebb1a8e07f790da3e459a089802eb5>

---
### Example 3

**Auto Label:** Improper state mutation leads to fund loss, incorrect balance tracking, and unauthorized claim ordering due to failed recovery or missing withdrawal mechanisms.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Medium

## Description

The `LidoStrategy` and `DSRStrategy` contracts are responsible for managing `ETH` and `DAI` deposits into `Lido` and `sDAI`, respectively, to earn yields.

These strategies are integrated with the `DepositETH` and `DepositUSD` contracts, which handle user deposits and facilitate interactions with `Lido` and `sDAI`.

```solidity
function executeStrategies(StrategyExecution[] memory _executionData) external override {
    if(msg.sender!=strategyExecutor) revert IncorrectStrategyExecutor(msg.sender);
    unchecked{
        for(uint i=0;i<_executionData.length;i++){
            if(!IStrategyManager(strategyManager)
                .strategyExists(_executionData[i].strategy)) revert IncorrectStrategy(_executionData[i].strategy);
@>          (bool success, bytes memory returndata) =
                _executionData[i].strategy.delegatecall(_executionData[i].executionData);
            if(!success){
                if (returndata.length == 0) revert CallFailed();
                assembly {
                    revert(add(32, returndata), mload(returndata))
                }
            }
---
}
```

However, the `LidoStrategy` and `DSRStrategy` contracts lack withdrawal functionality. As a result, the `DepositETH` and `DepositUSD` contracts cannot retrieve deposited tokens along with their accumulated rewards.

This limitation prevents the protocol from redeeming assets when needed, leaving it unable to maintain reserves required to fulfill users’ share redemptions, ultimately resulting in users losing access to their funds.

## Recommendations

Implement withdrawal functionality to the `LidoStrategy` and `DSRStrategy` contracts, enabling the protocol to retrieve deposited tokens along with accumulated rewards.

Moreover, for the `Lido` strategy, the withdrawal from Lido (primary market) requires the request and claim process to integrate with their withdrawal queue so the protocol will need to handle the withdrawal request and claim process or choose to integrate withdrawal with the secondary market.

---
