# Cluster -1055

**Rank:** #376  
**Count:** 14  

## Label
Unauthorized access to critical configuration or state parameters, enabling attackers to manipulate market operations, trigger malicious callbacks, or bypass access controls and sanctions.

## Cluster Information
- **Total Findings:** 14

## Examples

### Example 1

**Auto Label:** Unauthorized access to critical configuration or state parameters, enabling attackers to manipulate market operations, trigger malicious callbacks, or bypass access controls and sanctions.  

**Original Text Preview:**

##### Description
This issue has been identified within the [changeMarketMakerAddress](https://github.com/longgammalabs/hanji-contracts/blob/70b15ec4d9e7578248141604503843716a67d875/src/HanjiLOB.sol#L313) function of the `HanjiLOB` contract.

The function allows the administrator to change the market maker address and includes a flag, `_should_invoke_on_trade`, that determines whether callbacks are invoked after each trade for the `marketmaker` address. However, the function also permits anyone to change the market maker address if the current market maker address is set to zero. This presents a security risk as it could enable any actor to set a malicious `marketmaker` address with the `_should_invoke_on_trade` flag set to `true`. This setup could allow them to engage in sandwich attacks and claim additional fees, thereby manipulating market operations.

The issue is classified as **medium** severity because it introduces a potential vulnerability that could be exploited to interfere with the market's integrity.
##### Recommendation
We recommend restricting the ability to change the market maker address exclusively to the owner of the contract, even if the current market maker address is set to zero.

---
### Example 2

**Auto Label:** Unauthorized access to critical configuration or state parameters, enabling attackers to manipulate market operations, trigger malicious callbacks, or bypass access controls and sanctions.  

**Original Text Preview:**

##### Description
This issue has been identified within the [changeMarketMakerAddress](https://github.com/longgammalabs/hanji-contracts/blob/70b15ec4d9e7578248141604503843716a67d875/src/HanjiLOB.sol#L313) function of the `HanjiLOB` contract.

The function allows the administrator to change the market maker address and includes a flag, `_should_invoke_on_trade`, that determines whether callbacks are invoked after each trade for the `marketmaker` address. However, the function also permits anyone to change the market maker address if the current market maker address is set to zero. This presents a security risk as it could enable any actor to set a malicious `marketmaker` address with the `_should_invoke_on_trade` flag set to `true`. This setup could allow them to engage in sandwich attacks and claim additional fees, thereby manipulating market operations.

The issue is classified as **medium** severity because it introduces a potential vulnerability that could be exploited to interfere with the market's integrity.
##### Recommendation
We recommend restricting the ability to change the market maker address exclusively to the owner of the contract, even if the current market maker address is set to zero.

---
### Example 3

**Auto Label:** Unauthorized access to critical configuration or state parameters, enabling attackers to manipulate market operations, trigger malicious callbacks, or bypass access controls and sanctions.  

**Original Text Preview:**

##### Description
This issue has been identified within the [changeMarketMakerAddress](https://github.com/longgammalabs/hanji-contracts/blob/70b15ec4d9e7578248141604503843716a67d875/src/HanjiLOB.sol#L313) function of the `HanjiLOB` contract.

The function allows the administrator to change the market maker address and includes a flag, `_should_invoke_on_trade`, that determines whether callbacks are invoked after each trade for the `marketmaker` address. However, the function also permits anyone to change the market maker address if the current market maker address is set to zero. This presents a security risk as it could enable any actor to set a malicious `marketmaker` address with the `_should_invoke_on_trade` flag set to `true`. This setup could allow them to engage in sandwich attacks and claim additional fees, thereby manipulating market operations.

The issue is classified as **medium** severity because it introduces a potential vulnerability that could be exploited to interfere with the market's integrity.
##### Recommendation
We recommend restricting the ability to change the market maker address exclusively to the owner of the contract, even if the current market maker address is set to zero.

---
