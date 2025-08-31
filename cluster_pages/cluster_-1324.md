# Cluster -1324

**Rank:** #366  
**Count:** 16  

## Label
Premature exits and flawed validation lead to missed callbacks, cascading failures, and unauthorized fee drainage, compromising oracle integrity and protocol reliability.

## Cluster Information
- **Total Findings:** 16

## Examples

### Example 1

**Auto Label:** Failure to validate or properly handle ETH fees in oracle calls, leading to fund loss, transaction reverts, or unauthorized ether depletion.  

**Original Text Preview:**

##### Description
In the `OracleRequestRecipient.handle()` function the `msg.value` parameter is missing [in the call](https://github.com/diadata-org/Spectra-interoperability/blob/ed9f1e5ff3aa6cfba02d12f0bed1e435aeec24c1/contracts/contracts/OracleRequestRecipient.sol#L76) to `OracleTrigger.dispatch()`. As a result, `IMailbox(mailBox).dispatch()` will always revert, as the passed `msg.value` will be zero.

Also for the `OracleRequestRecipient.handle()` call to be executed with `msg.value > 0`, `Mailbox.process()` must also be called with `msg.value > 0`. This raises the question of how to ensure that relayers operate with an additional `msg.value`. It may be necessary to run custom relayers, but this approach seems suboptimal.

This issue has been assigned a **High** severity level because it breaks the core functionality of the `OracleRequestRecipient` contract, and the only way to fix it would be through a redeployment.

##### Recommendation
We recommend adding the `msg.value` to the call:
```solidity
IOracleTrigger(oracleTriggerAddress).dispatch{value: msg.value}(
    _origin, sender, key);
```  

Additionally, we recommend developing a mechanism to paying the fee when calling `OracleRequestRecipient.handle()`.

---
### Example 2

**Auto Label:** Failure to validate or properly handle ETH fees in oracle calls, leading to fund loss, transaction reverts, or unauthorized ether depletion.  

**Original Text Preview:**

##### Description

The `PythAggregator` contract's `updatePrice` function, which is called indirectly through `getPrice()`, does not refund excess ETH sent by users. This function is called in multiple payable functions across `Router.sol`, `Factory.sol`, and `Vault.sol`, allowing users to provide `msg.value`. The relevant code in `PythAggregator.sol` is:

```
function updatePrice(bytes[] calldata _priceUpdateData) public payable {
    uint256 fee = pyth.getUpdateFee(_priceUpdateData);
    pyth.updatePriceFeeds{value: fee}(_priceUpdateData);
}
```

This implementation transfers the exact fee to the `Pyth` contract but does not refund any excess ETH sent by the user. The excess ETH remains trapped in the `PythAggregator` contract.

Users who inadvertently send more ETH than required for price updates will lose their excess funds. This results in a direct financial loss for users and accumulation of unusable ETH in the `PythAggregator` contract. The severity of this issue is heightened by the fact that it affects multiple core functions of the protocol in Router.sol,Factory.sol and of course Vault.sol

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:N/A:N/D:M/Y:N/R:N/S:C (6.3)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:N/A:N/D:M/Y:N/R:N/S:C)

##### Recommendation

It is recommended to implement a refund mechanism in the `updatePrice` function to return any excess ETH to the user. Here's an example of how to modify the function:

```
function updatePrice(bytes[] calldata _priceUpdateData) public payable {
    uint256 fee = pyth.getUpdateFee(_priceUpdateData);
    pyth.updatePriceFeeds{value: fee}(_priceUpdateData);
    uint256 excess = msg.value - fee;
    if (excess > 0) {
        (bool success, ) = msg.sender.call{value: excess}("");
        require(success, "ETH refund failed");
    }
}
```

##### Remediation

**SOLVED:** Excess fees are now refunded to `msg.sender.`

##### Remediation Hash

<https://github.com/prodigyfi/brt-dci-contracts/commit/d422f6cefa174580ce3afe6c03570cff8551149b>

##### References

[prodigyfi/brt-dci-contracts/src/PythAggregator.sol#L42](https://github.com/prodigyfi/brt-dci-contracts/blob/master/src/PythAggregator.sol#L42)

---
### Example 3

**Auto Label:** Failure to validate or properly handle ETH fees in oracle calls, leading to fund loss, transaction reverts, or unauthorized ether depletion.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

In the `fullfillOracleQueryPyth` function within `Pyth.sol`, there is a potential issue where users can inadvertently send more Ether (`msg.value`) than required to the `pyth.updatePriceFeeds` function. This overpayment has not been refunded, resulting in a loss of Ether.

The function implementation is as follows:

```solidity
function fullfillOracleQueryPyth(address pythAddress, bytes calldata signedOffchainData) {
    IPyth pyth = IPyth(pythAddress);

    (bytes[] memory updateData) = abi.decode(signedOffchainData, (bytes[]));

>>  try pyth.updatePriceFeeds{ value: msg.value }(updateData) { }
    catch (bytes memory reason) {
        if (_isFeeRequired(reason)) {
            revert Errors.FeeRequiredPyth(pyth.getUpdateFee(updateData));
        } else {
            uint256 len = reason.length;
            assembly {
                revert(add(reason, 0x20), len)
            }
        }
    }
}
```

The `msg.value` is transferred without validating the exact fee required by `pyth.updatePriceFeeds`. Based on the `Pyth::updatePriceFeeds` function at the address [`0xdd24f84d36bf92c65f92307595335bdfab5bbd21`](https://etherscan.io/address/0xdd24f84d36bf92c65f92307595335bdfab5bbd21#code#F2#L71), there is no mechanism that returns any excess ETH sent:

```solidity
    function updatePriceFeeds(
        bytes[] calldata updateData
    ) public payable override {
        uint totalNumUpdates = 0;
        for (uint i = 0; i < updateData.length; ) {
            if (
                updateData[i].length > 4 &&
                UnsafeCalldataBytesLib.toUint32(updateData[i], 0) ==
                ACCUMULATOR_MAGIC
            ) {
                totalNumUpdates += updatePriceInfosFromAccumulatorUpdate(
                    updateData[i]
                );
            } else {
                updatePriceBatchFromVm(updateData[i]);
                totalNumUpdates += 1;
            }

            unchecked {
                i++;
            }
        }
        uint requiredFee = getTotalFee(totalNumUpdates);
        if (msg.value < requiredFee) revert PythErrors.InsufficientFee();
    }
```

Therefore, two scenarios could occur:

- A user sends more ETH than expected, resulting in the extra ETH not being returned to the user by `Pyth`.
- Since the `Pyth` contract is upgradeable, [the fees in `pyth` could change](https://etherscan.io/address/0xdd24f84d36bf92c65f92307595335bdfab5bbd21#code#F2#L62) before the function `fullfillOracleQueryPyth` is executed, causing them to differ from what was expected.

## Recommendations

Before calling `pyth.updatePriceFeeds`, calculate and validate the exact fee required. This can be achieved by calling `pyth.getUpdateFee(updateData)` and comparing it with `msg.value`.

```diff
function fullfillOracleQueryPyth(address pythAddress, bytes calldata signedOffchainData) {
    IPyth pyth = IPyth(pythAddress);

    (bytes[] memory updateData) = abi.decode(signedOffchainData, (bytes[]));
+   uint fee = pyth.getUpdateFee(updateData);
+   require(msg.value == fee);
    try pyth.updatePriceFeeds{ value: msg.value }(updateData) { }
    catch (bytes memory reason) {
        if (_isFeeRequired(reason)) {
            revert Errors.FeeRequiredPyth(pyth.getUpdateFee(updateData));
        } else {
            uint256 len = reason.length;
            assembly {
                revert(add(reason, 0x20), len)
            }
        }
    }
}
```

---
