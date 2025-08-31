# Cluster -1042

**Rank:** #123  
**Count:** 126  

## Label
Inadequate test coverage and validation lead to undetected vulnerabilities in core contract logic, exposing risks of exploitable behavior under edge cases or real-world conditions.

## Cluster Information
- **Total Findings:** 126

## Examples

### Example 1

**Auto Label:** Inadequate test coverage and validation lead to undetected vulnerabilities in core contract logic, exposing risks of exploitable behavior under edge cases or real-world conditions.  

**Original Text Preview:**

Throughout the codebase, and in particular in the added changes, multiple instances of insufficient test coverage were identified:

* There are different test suites implemented at the same time, namely Hardhat and Foundry. Maintaining different suites instead of having all under the same one adds friction and error-proneness, and increases the cost for the developer to keep it secure.
* Similar contracts in the protocol use different test suites. In particular, [Universal contracts](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/test/evm/foundry/local/Universal_Adapter.t.sol#L16) rely on Foundry, while the [Arbitrum](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/test/evm/hardhat/chain-adapters/Arbitrum_Adapter.ts#L59) contracts rely on Hardhat. Standardizing the tests would allow for testing similar contracts under the same cases, which could be beneficial when finding edge cases or bugs.
* New additions only add 5 single positive overall cases to the suite, leaving many other edge cases untested and not asserting any negative situations.
* The values used for testing the outgoing fees have been [set to zero](https://github.com/across-protocol/contracts/blob/c5d7541037d19053ce2106583b1b711037483038/test/evm/foundry/local/Universal_Adapter.t.sol#L291), as a result of which the whole fee protection is bypassed.
* Contracts are not fuzzed to find edge cases that could be used for exploits.
* There is a lack of integration with the LayerZero protocol, which requires analyzing its caveats, edge cases, and behaviors, and testing the project under such conditions.

Insufficient testing, while not a specific vulnerability, implies a high probability of additional undiscovered vulnerabilities and bugs. It also exacerbates multiple interrelated risk factors in a complex code base. This includes a lack of complete, implicit specification of the functionality and exact expected behaviors that tests normally provide, which increases the chances of correctness issues being missed. It also requires more effort to establish basic correctness and reduces the effort spent exploring edge cases, thereby increasing the chances of missing complex issues.

Moreover, the lack of repeated automated testing of the full specification increases the chances of introducing breaking changes and new vulnerabilities. This applies to both previously audited code and future changes to currently audited code. Underspecified interfaces and assumptions increase the risk of subtle integration issues which testing could reduce by enforcing an exhaustive specification.

To address these issues, consider implementing a comprehensive multi-level test suite. Such a test suite should comprise contract-level tests with 95%-100% coverage, per chain/layer deployment, and integration tests that test the deployment scripts as well as the system as a whole, along with per chain/layer fork tests for planned upgrades. Crucially, the test suite should be documented in such a way that a reviewer can set up and run all these test layers independently of the development team. Some existing examples of such setups can be suggested for use as reference in a follow-up conversation. In addition, consider merging all the test suites into a single one for better maintenance. Implementing such a test suite should be of very high priority to ensure the system's robustness and reduce the risk of vulnerabilities and bugs.

***Update:** Partially resolved in [pull request #1038](https://github.com/across-protocol/contracts/pull/1038). The team stated:*

> *Addressed these 2 points:*
>
> *- New additions only add 5 single positive overall cases to the suite, leaving many other edge cases untested and not asserting any negative situations.*
>
> *- The values used for testing the outgoing fees have been set to zero, as a result of which the whole fee protection is bypassed.*
>
> *Added multiple new local (unit) test-cases (fee ones were added as part of the fix for issue M-03) as well as a fork test for sending USDT via a `Universal_Adapter`. A fork test can be run with this command:*
>
> *`NODE_URL_1=<your-ethereum-rpc-url> forge test --match-path test/evm/foundry/fork/UniversalAdapterOFT.t.sol`*

---
### Example 2

**Auto Label:** Inadequate test coverage and validation lead to undetected vulnerabilities in core contract logic, exposing risks of exploitable behavior under edge cases or real-world conditions.  

**Original Text Preview:**

## Diﬃculty: High

## Type: Auditing and Logging

## Description

Code that was added to `reserve-index-dtf` to support trusted fillers and rebalancing is not sufficiently tested (figures 1.1–1.3). Several new functions in the Folio contract and the TrustedFillerRegistry contract are not tested at all, and some functions in the CowSwapFiller contract are only partially tested. Insufficient testing of code could allow bugs to go unnoticed.

### Figures
- **Figure 1.1**: Combined test coverage of the `reserve-index-dtf` and `trusted-fillers` repositories
- **Figure 1.2**: Test coverage of the `TrustedFillerRegistry` contract
- **Figure 1.3**: Test coverage of the `CowSwapFiller` contract and `GPv2OrderLib` library

The following Folio functions are not tested at all:
- `stateChangeActive`
- `setTrustedFillerRegistry`
- `getRebalance`
- `endRebalance`

Note that the lack of testing of `endRebalance` implies that there is no test that `startRebalance` can be called following a call to `endRebalance`. This is a critical condition that the tests should cover.

There are also no tests to verify proper handling of nonces. More specifically, each Folio deployment holds a `Rebalance` struct with a nonce. Folio functions that act on an Auction check that its nonce matches the Rebalance’s nonce and revert if there is a mismatch. However, there are currently no tests to verify that Auction-acting functions revert when expected to.

The following TrustedFillerRegistry functions are not tested at all:
- `deprecateTrustedFiller`
- `isAllowed`

The `swapActive` and `rescueToken` functions in the `CowSwapFiller` contract are only partially tested (figure 1.4). First, `swapActive`’s `block.number != blockInitialized` branch is not tested. Second, while `swapActive` and `rescueToken` are called internally, they are not called externally.

> The low coverage numbers for the `GPv2OrderLib` library are not concerning. The functions in that library are tested in the `cowprotocol/contracts` repository.

**Figure 1.4**: Excerpt of the `CowSwapFiller` contract’s test coverage

Furthermore, while there is a test of `isValidSignature`, there is no test of a complete interaction with the Cow Swap contracts. A function could appear correct when tested in isolation, yet it could fail when integrated into a larger system.

## Exploit Scenario

An exploitable bug is discovered in the trusted-fillers code. The bug could have been uncovered through more thorough testing.

## Recommendations

**Short term**, take the following steps:
- Add tests to the `reserve-index-dtf` or `trusted-fillers` repositories so that 100% statement coverage of the code added to support the new features is achieved.
- Add tests for the scenarios mentioned above:
  - A call to `endRebalance` followed by a call to `startRebalance`
  - Nonce handling (e.g., that Auctions with invalid nonces are rejected)
- Develop a `MockCowSwap` contract to simulate the actual CoW Swap contract, and use the mock contract in tests.

Taking these steps will help to ensure that the code added to support the new features is free of bugs.

**Long term**, regularly review the project’s test coverage. If a critical condition is not tested despite 100% statement coverage, add a test for that condition. Doing so will provide further assurance that the code is free of bugs.

---
### Example 3

**Auto Label:** Inadequate test coverage and validation lead to undetected vulnerabilities in core contract logic, exposing risks of exploitable behavior under edge cases or real-world conditions.  

**Original Text Preview:**

As the protocol expands, it is crucial to maintain a comprehensive testing suite that covers all the new features. Presently, the repository does not contain any unit tests for the [`ERC7683OrderDepositor`](https://github.com/across-protocol/contracts/blob/401e24ccca1b3af919dd521e58acd445297b65b6/contracts/erc7683/ERC7683OrderDepositor.sol#L20) and [`ERC7683OrderDepositorExternal`](https://github.com/across-protocol/contracts/blob/401e24ccca1b3af919dd521e58acd445297b65b6/contracts/erc7683/ERC7683OrderDepositorExternal.sol#L16) contracts.

In order to improve the robustness and security of the codebase, consider implementing a comprehensive testing suite for the EIP-7683 implementation with a high degree of coverage.

***Update:** Acknowledged, will resolve. The team is planning to integrate changes into the testing and coverage and the issue will be resolved with them.*

---
