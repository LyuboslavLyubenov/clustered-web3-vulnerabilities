# Cluster -1319

**Rank:** #281  
**Count:** 29  

## Label
Premature state modifications and gas-limited transfers create race conditions and transaction failures, leading to inefficiencies, logical errors, and operational risks in token handling and state transitions.

## Cluster Information
- **Total Findings:** 29

## Examples

### Example 1

**Auto Label:** Premature state modifications and gas-limited transfers create race conditions and transaction failures, leading to inefficiencies, logical errors, and operational risks in token handling and state transitions.  

**Original Text Preview:**

##### Description
This issue has been identified within the [_placeOrder](https://github.com/longgammalabs/hanji-contracts/blob/70b15ec4d9e7578248141604503843716a67d875/src/HanjiLOB.sol#L1001) and [_handleTokenTransfer](https://github.com/longgammalabs/hanji-contracts/blob/70b15ec4d9e7578248141604503843716a67d875/src/HanjiLOB.sol#L1457) functions of the `HanjiLOB` contract.

These functions use the `transfer` method inside to handle the native ETH transfers. The `transfer` function restricts the gas available for the receiving contract's function to 2300 gas. If the receiver has any logic in its `fallback` or `receive` function that requires more than 2300 gas, the transaction will revert. This limitation can lead to unexpected failures and disrupt the contract's operation, especially when interacting with complex contracts.

The issue is classified as **low** severity because it does not pose an immediate risk but can lead to transaction failures in specific scenarios involving more complex receiving contracts.
##### Recommendation
We recommend replacing the use of `transfer` with `call`, which doesn't limit the amount of gas to forward. This approach is safer and more flexible, accommodating receivers that may need more than 2300 gas to execute their logic.

Example:
```solidity!
(bool success, ) = msg.sender.call{value: msg.value}("");
require(success, "Transfer failed");
```

***

---
### Example 2

**Auto Label:** Premature state modifications and gas-limited transfers create race conditions and transaction failures, leading to inefficiencies, logical errors, and operational risks in token handling and state transitions.  

**Original Text Preview:**

##### Description
This issue has been identified within the [_placeOrder](https://github.com/longgammalabs/hanji-contracts/blob/70b15ec4d9e7578248141604503843716a67d875/src/HanjiLOB.sol#L998) function of the `HanjiLOB` contract. 

The function processes WETH deposits based on the `msg.value` passed in the transaction. However, the `weth.deposit{value: actual_value}()` operation occurs before the completion of all transfer-related operations. This can lead to a potential redundant `weth.deposit` followed by a subsequent `weth.withdraw` operation within the `_handleTokenTransfer` function if the `msg.value` exceeds the required amount of tokens to be sent to the contract. Such inefficiencies can lead to unnecessary gas consumption and complicate the logic.

The issue is classified as **low** severity because, while it does not immediately threaten contract functionality, it can lead to inefficiencies and potential logical errors in the contract's operation.
##### Recommendation
We recommend moving the `weth.deposit` operation to the `_handleTokenTransfer` function to ensure that all necessary validations and operations are successfully completed before the deposit is made.

---
### Example 3

**Auto Label:** Premature state modifications and gas-limited transfers create race conditions and transaction failures, leading to inefficiencies, logical errors, and operational risks in token handling and state transitions.  

**Original Text Preview:**

##### Description
This issue has been identified within the [_placeOrder](https://github.com/longgammalabs/hanji-contracts/blob/70b15ec4d9e7578248141604503843716a67d875/src/HanjiLOB.sol#L1001) and [_handleTokenTransfer](https://github.com/longgammalabs/hanji-contracts/blob/70b15ec4d9e7578248141604503843716a67d875/src/HanjiLOB.sol#L1457) functions of the `HanjiLOB` contract.

These functions use the `transfer` method inside to handle the native ETH transfers. The `transfer` function restricts the gas available for the receiving contract's function to 2300 gas. If the receiver has any logic in its `fallback` or `receive` function that requires more than 2300 gas, the transaction will revert. This limitation can lead to unexpected failures and disrupt the contract's operation, especially when interacting with complex contracts.

The issue is classified as **low** severity because it does not pose an immediate risk but can lead to transaction failures in specific scenarios involving more complex receiving contracts.
##### Recommendation
We recommend replacing the use of `transfer` with `call`, which doesn't limit the amount of gas to forward. This approach is safer and more flexible, accommodating receivers that may need more than 2300 gas to execute their logic.

Example:
```solidity!
(bool success, ) = msg.sender.call{value: msg.value}("");
require(success, "Transfer failed");
```

***

---
