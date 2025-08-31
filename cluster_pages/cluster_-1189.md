# Cluster -1189

**Rank:** #22  
**Count:** 704  

## Label
Inadequate documentation and poor code clarity introduce exploitable logic gaps, misinterpretations, and implementation ambiguities, increasing risks of security flaws and errors during audits or development.

## Cluster Information
- **Total Findings:** 704

## Examples

### Example 1

**Auto Label:** Inadequate documentation and poor code clarity introduce exploitable logic gaps, misinterpretations, and implementation ambiguities, increasing risks of security flaws and errors during audits or development.  

**Original Text Preview:**

Throughout the codebase, multiple instances of missing docstrings were identified. Particularly, in the following files:

* `AdapterStore.sol`
* `Arbitrum_Adapter.sol`
* `Arbitrum_SpokePool.sol`
* `OFTTransportAdapter.sol`
* `OFTTransportAdapterWithStore.sol`
* `Ovm_SpokePool.sol`
* `PolygonZkEVM_SpokePool.sol`
* `Polygon_SpokePool.sol`
* `Scroll_SpokePool.sol`
* `SpokePool.sol`
* `Succinct_SpokePool.sol`
* `Universal_Adapter.sol`
* `Universal_SpokePool.sol`
* `ZkSync_SpokePool.sol`

Consider thoroughly documenting all functions (and their parameters) that are part of any contract's public API, events, storage variables, and constants. Functions implementing sensitive functionality, even if not public, should be clearly documented as well. When writing docstrings, consider following the [Ethereum Natural Specification Format](https://solidity.readthedocs.io/en/latest/natspec-format.html) (NatSpec).

***Update:** Partially resolved at commit [84f87be](https://github.com/across-protocol/contracts/commit/84f87beed89c427d45e1c1ef7a0080d1e82f94ba). The team stated:*

> *Added docstrings to:*
>
> *- `AdapterStore.sol`* *- `OFTTransportAdapter.sol`* *- `OFTTransportAdapterWithStore.sol`*
>
> *to keep changes to the OFT scope.*

---
### Example 2

**Auto Label:** Inadequate documentation and poor code clarity introduce exploitable logic gaps, misinterpretations, and implementation ambiguities, increasing risks of security flaws and errors during audits or development.  

**Original Text Preview:**

Throughout the codebase, multiple instances of misleading documentation were identified:

* In the `Universal_Adapter` contract, the [inline documentation states](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/contracts/chain-adapters/Universal_Adapter.sol#L74) that the `relayTokens` function *"only uses the `CircleCCTPAdapter` to relay USDC tokens to CCTP enabled L2 chains"*. While this is partially true for that token, the new implementation also allows for using the [OFT method](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/contracts/chain-adapters/Universal_Adapter.sol#L90-L92).
* A [comment](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/contracts/AlephZero_SpokePool.sol#L3) in the `AlephZero_SpokePool` contract states that *" Arbitrum only supports v0.8.19"*. However, on Arbiscan, newer versions are also [listed](https://arbiscan.io/solcversions) as being supported.

Consider updating the documentation to reflect the current behavior of the functionality.

***Update:** Partially resolved in [pull request #1030](https://github.com/across-protocol/contracts/pull/1030). Only the `Universal_Adapter` contract has been updated.*

---
### Example 3

**Auto Label:** Inadequate documentation and poor code clarity introduce exploitable logic gaps, misinterpretations, and implementation ambiguities, increasing risks of security flaws and errors during audits or development.  

**Original Text Preview:**

**Description:** As part of changes in this audit, the `latestPrizeId` variable now functions as a counter for the next available prize ID, rather than tracking the most recently used one as in the previous version. This creates a naming mismatch, since `latestPrizeId` no longer reflects its actual behavior.

The function `_addPrizes` even caches it as `nextPrizeId`, which is a more accurate description. Consider renaming it as `nextPrizeId`.

**Linea:** Fixed in commits [`8266a91`](https://github.com/Consensys/linea-hub/pull/555/commits/8266a9130db2e3ad9d66422c7c1c374dc806bada) and [`dd8de7b`](https://github.com/Consensys/linea-hub/pull/555/commits/dd8de7bd9626abca6c01bc24832551c8f50ef0a9)

**Cyfrin:** Verified. Field renamed to `nextPrizeId` and stack variable in `__addPrizes` renamed to `currentPrizeId`.

\clearpage

---
