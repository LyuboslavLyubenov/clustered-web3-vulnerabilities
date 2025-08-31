# Cluster -1290

**Rank:** #424  
**Count:** 10  

## Label
Missing upper bounds on administrative fee settings enable arbitrary, unbounded fee increases, leading to excessive costs, financial loss, and degraded user trust due to unchecked administrative control.

## Cluster Information
- **Total Findings:** 10

## Examples

### Example 1

**Auto Label:** Missing upper bounds on administrative fee settings enable arbitrary, unbounded fee increases, leading to excessive costs, financial loss, and degraded user trust due to unchecked administrative control.  

**Original Text Preview:**

##### Description
An admin currently has the ability to set fees up to **99.99%**. This introduces the risk of excessive fees that may discourage platform use and harm users.

https://github.com/Liquorice-HQ/contracts/blob/a5b4c6a56df589b8ea4f6c7b8cb028b1723ad479/src/contracts/Repository.sol#L310-L324
##### Recommendation
We recommend implementing a lower maximum fee boundary and introducing a gradual, time-based increase for fee values. This approach will help protect users from unexpected or excessively high fees and ensure a more predictable cost structure.

---
### Example 2

**Auto Label:** Missing upper bounds on administrative fee settings enable arbitrary, unbounded fee increases, leading to excessive costs, financial loss, and degraded user trust due to unchecked administrative control.  

**Original Text Preview:**

**Severity**: Medium	

**Status**: Acknowledged

**Description**

The `TokenBridgeBase.sol` smart contract implement some functions to define certain protocol fees:
```solidity
/**
    * @notice Set transfer fee for BRC-20
    * @param _brc20TransferFee Transfer fee for BRC-20 in satoshi
    */
   function setBrc20TransferFee(uint256 _brc20TransferFee) external onlyOwner {
       brc20TransferFee = _brc20TransferFee; 
       emit Brc20TransferFeeSet(_brc20TransferFee);
   }

   /**
    * @notice Set transfer fee for Runes
    * @param _runesTransferFee Transfer fee for Runes in satoshi
    */
   function setRunesTransferFee(uint256 _runesTransferFee) external onlyOwner {
       runesTransferFee = _runesTransferFee;  
       emit RunesTransferFeeSet(_runesTransferFee);
   }

   /**
    * @notice Set transfer fee for BTC
    * @param _btcTransferFee Transfer fee for BTC in satoshi
    */
   function setBtcTransferFee(uint256 _btcTransferFee) external onlyOwner {
       btcTransferFee = _btcTransferFee;  
       emit BtcTransferFeeSet(_btcTransferFee);
   }
```

However, there is no upper limit for the new fees to be set so they can be set up to any number, including really high ones.

**Recommendation**:

Define an upper limit for the new fees and add a check to ensure that the new value is lower or equal to the upper limit when executing the mentioned functions.


**Client comment**: 

Marking this as acknowledged, if there are no objections, as there is no intention of changing the behavior.

---
### Example 3

**Auto Label:** Missing upper bounds on administrative fee settings enable arbitrary, unbounded fee increases, leading to excessive costs, financial loss, and degraded user trust due to unchecked administrative control.  

**Original Text Preview:**

##### Description

- https://github.com/eywa-protocol/eywa-clp/blob/d68ba027ff19e927d64de123b2b02f15a43f8214/contracts/PortalV2.sol#L89
- https://github.com/eywa-protocol/eywa-clp/blob/d68ba027ff19e927d64de123b2b02f15a43f8214/contracts/Whitelist.sol#L90

The `PortalV2.unlock()` function charges the user a fee specified in the Whitelist contract, which can be increased by the admin up to 100%, affecting transactions already in progress based on lower fees. Currently, the user cannot set a threshold for the maximum fee, which can lead to situations where a user initiates a bridge transaction expecting a nominal fee, but the admin - without ill intent - increases the fee, resulting in the user receiving less money than anticipated.

##### Recommendation

There are various ways to address this issue. One option is to empower users to set a maximum fee before sending a transaction. Another option is to implement a mechanism to update the bridge fee with a notice period, such as one day, to alert users of the change.

---
