# Cluster -1255

**Rank:** #191  
**Count:** 60  

## Label
Inconsistent or missing validation of zero or invalid deposit amounts across asset types leads to unauthorized fund transfers, state corruption, or failed status updates, enabling front-running, misallocation, or permanent fund trapping.

## Cluster Information
- **Total Findings:** 60

## Examples

### Example 1

**Auto Label:** Failure to validate cross-chain transfer amounts due to unaccounted fees, dust removal, or decimal mismatches, enabling incorrect asset validation and potential unauthorized withdrawals.  

**Original Text Preview:**

In the `_transferViaOFT` function of the `OFTTransportAdapter` contract, the final verification [checks](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/contracts/libraries/OFTTransportAdapter.sol#L97) that the amount of assets passed as input matches the amount that will be received at destination. However, when invoking the messenger during the transfer, the [`_removeDust` function](https://docs.layerzero.network/v2/developers/evm/oft/quickstart#example) in LayerZero will be called, which will remove all those digits from the input amount that cannot be represented by the `sharedDecimals` value (by default, 6). This means that if the `_amount` input has a leftover after the integer division of the `decimalConversionRate` value, then the `amountReceivedLD` value will be lesser than the `_amount` value by that dust, so the two will not match.

Relatedly, there is an [inline comment](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/contracts/libraries/OFTTransportAdapter.sol#L69-L71) that states *" Setting `minAmountLD` equal to `amountLD` protects us from any changes to the sent amount due to internal OFT contract logic, e.g. `_removeDust`"*, referring to the [two values](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/contracts/libraries/OFTTransportAdapter.sol#L73-L74) being the same and equal to the amount sent, the `_amount` input. However, it is worth noting that this measure does not offer greater protection against dust removal since the latter is accomplished through the final [check on the `oftReceipt` output](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/contracts/libraries/OFTTransportAdapter.sol#L97). Indeed, this check only employs the amount sent (`_amount`) and not the minimum amount, but even if validation with the minimum amount is not performed, the dust removal protection would still be caught by the final check with the `oftReceipt` output. Even though the amount is being passed by the Dataworker when submitting the bundle, it should be noted that the input amount could be converted before sending it through the OFT messenger so that it does not contain any dust in the first place, minimizing the cases of reversion.

Consider implementing the necessary calculation and conversion to prevent dust from reverting the transaction. In particular, the amount could be rounded up to the next precision given by the `decimalConversionRate` value to prevent sending fewer assets than those passed by the Dataworker.

***Update:** Acknowledged, not resolved. The team stated:*

> *It's a design decision we went with: we added this rounding requirement to the relevant UMIP as a requirement for correctness of a bundle. So dataworker is responsible for providing correct decimals. Otherwise the bundle is deemed incorrect If there's a bug in dataworker code, OFT send will indeed revert; that's desired behavior.*

---
### Example 2

**Auto Label:** Failure to validate cross-chain transfer amounts due to unaccounted fees, dust removal, or decimal mismatches, enabling incorrect asset validation and potential unauthorized withdrawals.  

**Original Text Preview:**

The [`_transferViaOFT`](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/contracts/libraries/OFTTransportAdapter.sol#L57) function of the `OFTTransportAdapter` contract allows for using the OFT transport between chains. To assert that the assets have arrived correctly, the function implements a check that [compares](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/contracts/libraries/OFTTransportAdapter.sol#L97) the amount sent from one chain with the amount received on the other chain. However, if the underlying token charges fees on transfers, the two amounts will differ (i.e., the input `_amount` will not match the output `amountReceivedLD`).

Consider implementing the necessary logic to allow for the transfer of fee-on-transfer tokens. Alternatively, consider documenting the fact that fee-on-transfer tokens cannot be used with OFT transfers when the fees are enabled.

***Update:** Acknowledged, not resolved. The team stated:*

> *Yes, that's fine. We plan on only supporting reputable tokens without fees on transfer.*

---
### Example 3

**Auto Label:** Failure to validate cross-chain transfer amounts due to unaccounted fees, dust removal, or decimal mismatches, enabling incorrect asset validation and potential unauthorized withdrawals.  

**Original Text Preview:**

In the `OFTTransportAdapter` contract, the `_transferViaOFT` function [checks](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/contracts/libraries/OFTTransportAdapter.sol#L97) if the expected value received at the end matches the input passed at origin. In LayerZero, the [default value for the local decimals is 18](https://docs.layerzero.network/v2/developers/evm/oft/quickstart#constructing-an-oft-contract), but it can be changed. In such a scenario, the `amountReceivedLD` value will be expressed in local decimals at destination and it will not match the `_amount` input expressed in local decimals at origin. Consequently, the transfer will revert, and movement using the OFT mechanism in that combination will get stalled. Note that the movement will work if the decimals on both chains are the same.

Consider taking into account the difference in decimals on both chains and performing the conversions when validating the received amount.

***Update:** Acknowledged, not resolved. Assets that implement different decimals on source and destination, and therefore deviate from the [default implementation](https://github.com/LayerZero-Labs/LayerZero-v2/blob/88428755be6caa71cb1d2926141d73c8989296b5/packages/layerzero-v2/evm/oapp/contracts/oft/OFTCore.sol#L356), might override the `OFTAdapter._debit()` and `OFTCore._debitView()` functions, causing the reversion of the validations in the `_transferViaOFT` function. OFT implementations whose decimals are the same might not present this issue.*

*The team stated:*

> *`_transferViaOft` flow: We're interacting with OFTAdapter on chain. Here's [default implementation](https://github.com/LayerZero-Labs/LayerZero-v2/blob/88428755be6caa71cb1d2926141d73c8989296b5/packages/layerzero-v2/evm/oapp/contracts/oft/OFTAdapter.sol#L20) of that. We're calling (call is to `OFTAdapter` which inherits `OFTCore`):*
>
> *`OFTCore.send()` -> `OFTAdapter._debit()` -> `OFTCore._debitView()`*
>
> *This call chain starts [here](https://github.com/LayerZero-Labs/LayerZero-v2/blob/88428755be6caa71cb1d2926141d73c8989296b5/packages/layerzero-v2/evm/oapp/contracts/oft/OFTCore.sol#L173). Within `_debitView`, which is the default OFT implementation, we see [this](https://github.com/LayerZero-Labs/LayerZero-v2/blob/88428755be6caa71cb1d2926141d73c8989296b5/packages/layerzero-v2/evm/oapp/contracts/oft/OFTCore.sol#L349):*
>
> *`amountSentLD = _removeDust(_amountLD);`* *`amountReceivedLD = amountSentLD;`*
>
> *Amounts received and sent are both in the local decimals of the \*source chain(, so decimal discrepancies will not be a problem in the default OFT implementation. USDT0 uses the same underlying logic (although their [contracts](https://vscode.blockscan.com/ethereum/0xcd979b10a55fcdac23ec785ce3066c6ef8a479a4) are upgradeable).*
>
> *All in all, we don't expect to support tokens with non-standard decimal implementation in `_debit()` nor do we expect OApps to override this function in terms of decimals logic. What we might expect some OApps do is maybe override `_debit` in terms of adding extra fees, we won't be able to support those just yet.*

---
