# Cluster -1106

**Rank:** #440  
**Count:** 8  

## Label
Unbounded fee controls enable central authority to manipulate or disrupt core operations through arbitrary fee setting, leading to financial abuse, integer overflows, and denial-of-service.

## Cluster Information
- **Total Findings:** 8

## Examples

### Example 1

**Auto Label:** Unbounded fee controls enable central authority to manipulate or disrupt core operations through arbitrary fee setting, leading to financial abuse, integer overflows, and denial-of-service.  

**Original Text Preview:**

##### Description

The BridgeEVM contract contains a critical vulnerability in its implementation of the `minSend` check. The check is performed on the `amountWithFee` instead of the actual amount being sent, allowing for potential bypass of the minimum send limit.

The `_sendTokens` function checks the `minSend` requirement against `amountWithFee`:

```
if (minSend != 0 && amountWithFee < minSend) revert NotReachedMinLimit();
```

The actual amount sent is calculated as:

```
amount = amountWithFee - fee;
```

This implementation allows for a scenario where:

* `amountWithFee` equals `fee`
* `fee` is greater than or equal to `minSend`
* The resulting `amount` sent is zero

##### BVSS

[AO:A/AC:L/AX:L/R:N/S:U/C:N/A:N/I:N/D:N/Y:N (1.0)](/bvss?q=AO:A/AC:L/AX:L/R:N/S:U/C:N/A:N/I:N/D:N/Y:N)

##### Recommendation

Modify the `_sendTokens` function to check the `minSend` requirement against the actual amount being sent, not `amountWithFee`:

```
uint256 actualAmount = amountWithFee - fee;
if (minSend != 0 && actualAmount < minSend) revert NotReachedMinLimit();
```

##### Remediation

**SOLVED**: A new check was added which ensures that `minSend` is always greater than `fee:`

```
if (_minSend <= _fee) revert IncorrectMinSend();
```

##### Remediation Hash

bd69c7442c9987b1fd2d7a0d4cb2b23a75bb1fed

---
### Example 2

**Auto Label:** Unbounded fee controls enable central authority to manipulate or disrupt core operations through arbitrary fee setting, leading to financial abuse, integer overflows, and denial-of-service.  

**Original Text Preview:**

##### Description
The smart contract system of the bridge exhibits significant centralization and lack of trustlessness. The owner has the authority to change the implementation of both the `FireBridge` and `FToken` contracts. 

Minting of FBTC on the EVM chains, cross-chain EVM transfers, and spending the BTC locked at the Bitcoin-EVM bridge are also centralized.

Additionally, the owner can set unlimited fees, as the `FeeConfig.minFee` parameter is not capped.

Related code: https://github.com/fbtc-xyz/fbtc-contract/blob/d557e5e0b73eda5af86de5bf4431653056844e64/contracts/FeeModel.sol#L39

##### Recommendation
While centralization and trustfulness may be intended by design, it is recommended to implement constraints to cap the `minFee` parameter to prevent the setting of unlimited fees. This can help mitigate risks associated with excessive centralization and enhance the trustworthiness of the system.

Additionally, we recommend improving the validation infrastructure to bring more decentralization to the minting and spending process.

---
### Example 3

**Auto Label:** Abuse of fee mechanisms enables denial-of-service by exploiting per-operation costs or external recipient reverts, allowing attackers to monopolize or halt minting and disrupt fair access.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-04-titles-judging/issues/261 

The protocol has acknowledged this issue.

## Found by 
0rpse, ComposableSecurity, xiaoming90
## Summary

The minting process might be DOS by malicious fee recipients, breaking the protocol's core functionality. This would also result in the loss of fee for the rest of the innocent fee recipients as the minting cannot proceed, resulting in the minting fee not being collected.

## Vulnerability Detail

When minting a new token for a given work, the minting fee will be collected as per Line 236 below.

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L236

```solidity
File: Edition.sol
222:     /// @notice Mint a new token for the given work.
223:     /// @param to_ The address to mint the token to.
224:     /// @param tokenId_ The ID of the work to mint.
225:     /// @param amount_ The amount of tokens to mint.
226:     /// @param referrer_ The address of the referrer.
227:     /// @param data_ The data associated with the mint. Reserved for future use.
228:     function mint(
229:         address to_,
230:         uint256 tokenId_,
231:         uint256 amount_,
232:         address referrer_,
233:         bytes calldata data_
234:     ) external payable override {
235:         // wake-disable-next-line reentrancy
236:         FEE_MANAGER.collectMintFee{value: msg.value}(
237:             this, tokenId_, amount_, msg.sender, referrer_, works[tokenId_].strategy
238:         );
239: 
240:         _issue(to_, tokenId_, amount_, data_);
241:         _refundExcess();
242:     }
```

The collected minting fee will be routed or transferred to one or more of the following parties directly (Using a push mechanism):

1. Edition's creator (Using 0xSplit wallet)
2. Edition's attributions (no limit on the number of attributions in the current setup)  (Using 0xSplit wallet)
3. Collection referrer
4. Minting referrer
5. Protocol fee receiver

This approach introduced the risk of DOS to the minting process. As long as any of the parties above intentionally revert upon receiving the ETH minting fee, they could DOS the entire minting process of work.

Note: The parties using the 0xSplit wallet are not an issue because the wallet requires the recipients to claim the fees from the 0xSplit wallet manually (Using the pull mechanism instead)

## Impact

The minting process might be DOS by malicious fee recipients, breaking the protocol's core functionality. This would also result in the loss of fee for the rest of the innocent fee recipients as the minting cannot proceed, resulting in the minting fee not being collected.

## Code Snippet

https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/editions/Edition.sol#L236

## Tool used

Manual Review

## Recommendation

Consider adopting a pull mechanism for the fee recipients to claim their accrued fee instead of transferring the fee directly to them during the minting process.



## Discussion

**xiaoming9090**

Escalate

The two (2) actors mentioned in this report, the "Collection referrer" and "Minting referrer," are external parties of the protocol that are not fully trusted. When these two external actors behave maliciously, they could negatively impact the other innocent parties (Edition's creator, Edition's attributions, Protocol fee receiver), including the protocol, by preventing them from receiving their fee, resulting in a loss of assets for the victims.

As such, this issue should not be overlooked due to its risks and should be considered valid.

**sherlock-admin3**

> Escalate
> 
> The two (2) actors mentioned in this report, the "Collection referrer" and "Minting referrer," are external parties of the protocol that are not fully trusted. When these two external actors behave maliciously, they could negatively impact the other innocent parties (Edition's creator, Edition's attributions, Protocol fee receiver), including the protocol, by preventing them from receiving their fee, resulting in a loss of assets for the victims.
> 
> As such, this issue should not be overlooked due to its risks and should be considered valid.

You've created a valid escalation!

To remove the escalation from consideration: Delete your comment.

You may delete or edit your escalation comment anytime before the 48-hour escalation window closes. After that, the escalation becomes final.

**WangSecurity**

Agree with the escalation, even though it's pure griefing and the referrers lose fees themselves, the protocol also loses fee. Planning to accept and validate the report.

**Evert0x**

Result:
Medium
Has Duplicates

**sherlock-admin4**

Escalations have been resolved successfully!

Escalation status:
- [xiaoming9090](https://github.com/sherlock-audit/2024-04-titles-judging/issues/261/#issuecomment-2111482078): accepted

---
