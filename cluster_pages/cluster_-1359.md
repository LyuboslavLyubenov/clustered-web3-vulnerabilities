# Cluster -1359

**Rank:** #251  
**Count:** 37  

## Label
Improper handling of return data and state mutations leads to incorrect execution, replay attacks, and loss of funds due to flawed assumptions about EVM data persistence and error propagation.

## Cluster Information
- **Total Findings:** 37

## Examples

### Example 1

**Auto Label:** Improper handling of return data and state mutations leads to incorrect execution, replay attacks, and loss of funds due to flawed assumptions about EVM data persistence and error propagation.  

**Original Text Preview:**

## Vulnerability Report

## Severity: 
**Low Risk**

## Context: 
`CallbackHandler.sol#L51-L67`

## Description: 
Callbacks to the vault do not return any data, which means callbacks that expect a return value are not supported. This could limit the use cases of callbacks. For example:

- **Maker (or other ERC-3156 flash loan)** requires the `onFlashloan()` callback to return a specific hash value.
- **Uniswap V4** requires the `unlockCallback()` function to return data of type `bytes`. The call will revert if no data is returned due to decoding errors.
- **ERC-1155** requires the recipient to return a specific hash value when the `onERC1155Received()` and `onERC1155BatchReceived()` functions are called.

## Recommendation: 
Consider allowing the guardian to set the return data of a callback. For example, the return data can be stored during the `_allowCallback()` function and returned in the `_handleCallback()` function.

## Aera: 
Fixed in PR 293 and PR 356.

## Spearbit: 
Verified.

---
### Example 2

**Auto Label:** Improper handling of return data and state mutations leads to incorrect execution, replay attacks, and loss of funds due to flawed assumptions about EVM data persistence and error propagation.  

**Original Text Preview:**

The `StrategyManager.executeCall()` function is intended to be called by the owner to execute external calls, where it's primarily designed to allow users to manage their HyperLend positions:

```solidity
    function executeCall(
        address target,
        uint256 value,
        bytes memory data,
        bool allowRevert
    ) public onlyOwner returns (bytes memory) {
        (bool success, bytes memory returnData) = target.call{value: value}(
            data
        );
        if (!allowRevert) require(success, "execution reverted");
        return returnData;
    }
```

The function has the `allowRevert` parameter, where, if set to `true`, the failed call should **revert**, however, the current check implementation causes the call to revert when `allowRevert` is set to `false` when the owner doesn't intend for the call to revert, which contradicts the intended use of `allowRevert`.

As a result, if the owner executes a call and sets `allowRevert = true` (meaning the call should revert on failure), the transaction will not revert, and any changes made by the call will not be rolled back, where this could lead to a loss of the sent value and undesired results or state changes, especially if the call is part of a sequence of calls executed by the owner.

```diff
    function executeCall(
        address target,
        uint256 value,
        bytes memory data,
        bool allowRevert
    ) public onlyOwner returns (bytes memory) {
        (bool success, bytes memory returnData) = target.call{value: value}(
            data
        );
-       if (!allowRevert) require(success, "execution reverted");
+       if (allowRevert) require(success, "execution reverted");

        return returnData;
    }
```

---
### Example 3

**Auto Label:** Improper validation of contract existence or transaction outcomes leads to permanent fund loss or unauthorized access due to insufficient state checks and lack of error handling.  

**Original Text Preview:**

## TokenPayAndSettleIncoming Functionality

## Context
File: `channel_manager.rs`  
Lines: 719-724  

## Description
The `TokenPayAndSettleIncoming` functionality processes incoming transfers from EVM. It implements multiple checks on the expected fields. However, one of these checks is insufficient. The EVM channel checks that the hashlock value is not `[0u8; 32]` before processing the transfer. This check must be hardened to ensure that the hashlock value corresponds to the expected hashlock. The current implementation can lead the plugin to process invalid transfers.

## Recommendation
Ensure that the hashlock in the EVM smart contract corresponds to the hashlock passed in the request parameters.

## Resolution
- **SatsBridge**: Fixed in commit `f3b213b4`.
- **Cantina Managed**: Fixed. The hashlock is now correctly checked.

---
