# Cluster -1086

**Rank:** #395  
**Count:** 12  

## Label
Misuse of deprecated ETH transfer methods leads to silent failures, reentrancy risks, and loss of funds due to lack of result validation and EVM upgrade incompatibility.

## Cluster Information
- **Total Findings:** 12

## Examples

### Example 1

**Auto Label:** Misuse of deprecated ETH transfer methods leads to silent failures, reentrancy risks, and loss of funds due to lack of result validation and EVM upgrade incompatibility.  

**Original Text Preview:**

**Description**

PlatformAuction.sol: function_currency Transfer().
Due to the Istanbul update, there were several changes provided to the EVM, which made .transfer() and .send() methods deprecated for the ETH transfer. It is highly recommended to use .call() functionality with mandatory result check or the built- in functionality of the Address contract from OpenZeppelin library.

**Recommendation**

Correct ETH sending functionality with the call() function.

**Re-audit comment**

Resolved.

Post-audit:
The deprecated ETH transfer function was replaced with the call() function.

---
### Example 2

**Auto Label:** Misuse of deprecated ETH transfer methods leads to silent failures, reentrancy risks, and loss of funds due to lack of result validation and EVM upgrade incompatibility.  

**Original Text Preview:**

**Description**

MyStaking.sol: function daoWithdraw().
MyShare.sol: function withdraw Token().
MyExchange.sol: function withdraw().
MyWrapper.sol: Lines 201, 345, 463, 504, 530.
StakingFactory.sol: function withdraw Token().
Due to the Istanbul update, there were several changes provided to the EVM, which made .transfer() and .send() methods deprecated for the ETH transfer. Thus, it is highly recommended to use .call() functionality with mandatory result check or the built-in functionality of the Address contract from OpenZeppelin library. Also, the contract has neither payable function nor receive() function implemented, thus it isn't supposed to store ETH on its balance.

**Recommendation**

Correct the ETH sending functionality and verify its necessity in the contract.

**Re-audit comment**

Resolved

---
### Example 3

**Auto Label:** Misuse of deprecated ETH transfer methods leads to silent failures, reentrancy risks, and loss of funds due to lack of result validation and EVM upgrade incompatibility.  

**Original Text Preview:**

**Description**

LimitOrder ProtocolRFQ.sol: function fillOrderRFQTo(), line 165. Due to the Istanbul update there were several changes provided to the EVM, which made .transfer() and .send() methods deprecated for the ETH transfer. Thus it is highly recommended to use .call() functionality with mandatory result check, or the built-in functionality of the Address contract from OpenZeppelin library.

**Recommendation**

Correct ETH sending functionality.

**Re-audit comment**

Resolved.
Post-audit
Fixed after the 1st audit iteration

---
