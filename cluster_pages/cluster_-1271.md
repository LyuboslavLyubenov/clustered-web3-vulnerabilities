# Cluster -1271

**Rank:** #370  
**Count:** 15  

## Label
Hardcoded parameters in cross-chain interactions disable critical fee routing and flexibility, leading to denial-of-service, failed transactions, and loss of composability in LayerZero integrations.

## Cluster Information
- **Total Findings:** 15

## Examples

### Example 1

**Auto Label:** Misrouting of LayerZero fees leads to unauthorized refunds, fee misallocation, or loss of user funds via incorrect recipient addresses in cross-chain messaging.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

The LayerZero fee, which is intended to cover the cross-chain messaging cost for deposits and withdrawals, is not refunded to the sender who initiates the transaction through the Ethereum mainnet or Berachain.

Instead, this fee is directed to the deposit contract itself (`DepositETH`, `DepositUSD`, `DepositETHBera`, and `DepositUSDBera`), despite the fact that the sender is the one incurring the cost of the cross-chain messaging.

This can result in an unfair burden on the sender, who bears the full cost of initiating the cross-chain transaction without receiving any compensation for the excess fee.

```solidity
@> function sendMessage(bytes memory _data, uint32 _destId, uint256 _lzFee) external override payable onlyDeposit{
    if(!destIdAvailable(_destId)) revert NotWhitelisted(_destId);
    MessagingReceipt memory receipt = _lzSend(
        _destId,
        _data,
        optionsDestId[_destId],
        MessagingFee(_lzFee, 0),
@>        payable(msg.sender)
    );
    emit MessageSent(_destId,_data,receipt);
}
```

Using the `Messaging` contract as an example, the `sendMessage()` can only be called by `DepositETH` or `DepositUSD`. Consequently, the `msg.sender` passed as the refund address is either the `DepositETH` or `DepositUSD` contract, rather than the user who originally initiated the deposit.

To note that the `MessageBera.sendMessage()` that is used in `DepositETHBera` and `DepositUSDBera` contracts shares the same issues.

## Recommendations

Consider passing the user address that initiated the action as the refund address to ensure the fee is properly refunded to the user.

```diff
- function sendMessage(bytes memory _data, uint32 _destId, uint256 _lzFee) external override payable onlyDeposit{
+ function sendMessage(bytes memory _data, uint32 _destId, uint256 _lzFee, address refundAddress) external override payable onlyDeposit{
    if(!destIdAvailable(_destId)) revert NotWhitelisted(_destId);
    MessagingReceipt memory receipt = _lzSend(
        _destId,
        _data,
        optionsDestId[_destId],
        MessagingFee(_lzFee, 0),
-        payable(msg.sender)
+       payable(refundAddress)
    );
    emit MessageSent(_destId,_data,receipt);
}
```

---
### Example 2

**Auto Label:** Misrouting of LayerZero fees leads to unauthorized refunds, fee misallocation, or loss of user funds via incorrect recipient addresses in cross-chain messaging.  

**Original Text Preview:**

## Context
- LayerZeroAdapter.sol#L172
- LayerZeroAdapter.sol#L175

## Description
The `sendMessageUsingNative` function from the `LayerZeroAdapter.sol` contract is used by the `Bridge.sol` contract to send a message using LayerZero. This function, in turn, calls the `_sendMessage` function, which again calls `_lzSend` to call the LayerZero endpoint with the right parameters.

The `sendMessageUsingNative` function accepts `msg.value` equal to or greater than the fee quoted by LayerZero to cover LayerZero's messaging costs. It forwards them to the endpoint contract, from which any unused value is refunded to the `refundAddress` specified. 

However, these refunds don't work as expected as the refund address is always specified as the `Bridge.sol` contract, which is the `msg.sender` in the context of the `LayerZeroAdapter.sol` contract. Thus, any refunds received from LayerZero's endpoint are locked indefinitely in the `Bridge.sol` contract.

## Recommendation
The above-mentioned vulnerability could be fixed in multiple ways, including:
- Forwarding a user-specified refund address to LayerZero's endpoint to process refunds properly.
- Collecting all refunds from the endpoint to Bridge and refunding all the `msg.value` stored in `Bridge.sol` at the end of a bridging transaction back to the `msg.sender`.
- Making sure `msg.value` is strictly equal to the quoted fee, ensuring no refunds are possible.

## Sweep n' Flip
Fixed in snf-bridge-contracts-v1 PR 16.

## Cantina Managed
Verified fix. A new variable called `refundAddress` is forwarded from `Bridge.sol` to the LayerZero adapter, encoding `msg.sender` as the refund address.

---
### Example 3

**Auto Label:** Hardcoded zero fees bypass intended fee mechanisms, leading to disabled or misaligned revenue models and inconsistent cross-chain transaction behavior.  

**Original Text Preview:**

**Severity**: Low	

**Status**: Resolved

**Description**

When the `TokenBridgeBase.sol` smart contract is deployed, the `brc20TransferFee`, `runesTransferFee` and `btcTransferFee` are not being set, therefore they are 0.
```solidity
constructor(
       address _endpoint,
       address _btcBridge,
       address _superAdmin,
       address _btcDataFeed,
       address _nativeDataFeed,
       uint8 _nativeDecimals
   ) BridgeRoles(_superAdmin, _btcBridge) NonblockingLzApp(_endpoint) {
       btcDataFeed = AggregatorV3Interface(_btcDataFeed);
       nativeDataFeed = AggregatorV3Interface(_nativeDataFeed);
       nativeDecimals = _nativeDecimals;
   }
```

If the corresponding functions to modify these fees are not executed, then they will remain as 0.

**Recommendation**:

Define the values for the mentioned fees within the constructor. Do not forget to also define upper limits for these fees within the constructor.

---
