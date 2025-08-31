# Cluster -1345

**Rank:** #367  
**Count:** 16  

## Label
Failure to validate input parameters in swap logic leads to malformed transactions, unintended state changes, and potential fund loss due to unchecked assumptions and missing error handling in critical functions.

## Cluster Information
- **Total Findings:** 16

## Examples

### Example 1

**Auto Label:** Failure to properly validate or handle input conditions leads to permanent fund loss through incorrect state transitions, improper approvals, or unhandled reverts in swap and deposit logic.  

**Original Text Preview:**

## Context
`txnUtil.ts#L108-L117`

## Description
When performing a swap, the system generates two transactions: one to approve the required token amount and another to execute the swap. If the swap transaction fails, the approved token amount is not reset to zero. This results in the user's account retaining a token approval amount that does not align with the user's intentions, as the approval is meant only for the specific swap that failed.

## Recommendation
Consider implementing logic to reset the token approval to zero when the swap transaction fails. This ensures that the approval state in the user's account reflects their intended usage, avoiding unintended leftover approvals caused by system failures.

## ChainPro
This is left in place to make future transactions faster, given the immediate retry logic. The idea is to reuse the approval of the contract when swapping.

## Cantina Managed
Acknowledged by ChainPro team.

---
### Example 2

**Auto Label:** Missing input validation enables attackers to manipulate swap amounts, bypass collateral requirements, or trigger unauthorized partial/incorrect executions, leading to fund misallocation or rug pulls.  

**Original Text Preview:**

## Security Vulnerability in SwapFacility.sol

## Context
`SwapFacility.sol#L197`

## Description
The `swapExactCollateralForDebtAndLZSend` function allows malicious users to mint arbitrary amounts of debt tokens by manipulating LayerZero send parameters, bypassing collateral requirements. This issue exists because the function fails to validate that the cross-chain send amount (`sendParam.amountLD`) matches the calculated debt output (`debtOut`). 

### An attacker can:
- Provide a valid `collateralIn` amount.
- Set `sendParam.amountLD` to any arbitrary value (maximum pre-minted debt cap).
- Execute a cross-chain transfer with an inflated amount.
- Receive excess minted tokens on the destination chain.

## Proof of Concept
The following proof of concept demonstrates the issue:
- **ETH mainnet fork** (chainId: 1).
- **Arbitrum fork** (chainId: 42161).
- Deploys `BitcornOFT` on ETH and `WrappedBitcornNativeOFTAdapter` on Arbitrum:
  
```solidity
SendParam memory sendParam = SendParam(
    30110, // destination chain
    bytes32(uint256(uint160(deployer))), // recipient
    collateralIn, // amount to send - can be manipulated
    0, // minAmount
    OptionsBuilder.encodeLegacyOptionsType1(200_000),
    bytes(""),
    bytes("")
);
```

```solidity
swapFacility.swapExactCollateralForDebtAndLZSend{value: 1 ether}(
    1e18, // Only deposits 1 token as collateral
    sendParam, // But requests much larger amount in sendParam
    lzFee,
    deployer,
    block.timestamp
);
```

`sendParam.amountLD` is not validated against the actual collateral-backed debt amount (`debtOut`), allowing arbitrary amounts to be minted on the destination chain.

## Recommendation
Consider fixing the issue by following one of the following recommendations:
- Construct the `sendParam` inside the function rather than accept arbitrary inputs from the user; or...
- Validate the `sendParam` against the `debtOut` values to ensure the accuracy of the cross-chain message.

## Status
- **Bitcorn**: Fixed in f53bd74f.
- **Cantina Managed**: Verified fix. The `sendParam.amountLD` is now set to `debtOut` instead of the user-specified value, preventing this issue.

---
### Example 3

**Auto Label:** Failure to validate input parameters in swap logic leads to malformed transactions, unintended state changes, and potential fund loss due to unchecked assumptions and missing error handling in critical functions.  

**Original Text Preview:**

Whenever the codebase executes `abi.decode` in the [`Cell`](https://github.com/tesseract-protocol/smart-contracts/blob/940fc6650d1f48b3b3d028640dffcc9203f8bd76/src/Cell.sol#L52) or [`YakSwapCell`](https://github.com/tesseract-protocol/smart-contracts/blob/940fc6650d1f48b3b3d028640dffcc9203f8bd76/src/YakSwapCell.sol#L55) contract it is assuming that the decoding data is actually encoded correctly. This might be obvious in the `receiveFunction` since it is supposed to be called by a contract which is assumed to behaves correctly, but it might not be true in the [`route`](https://github.com/tesseract-protocol/smart-contracts/blob/940fc6650d1f48b3b3d028640dffcc9203f8bd76/src/YakSwapCell.sol#L58) and [`_swap`](https://github.com/tesseract-protocol/smart-contracts/blob/940fc6650d1f48b3b3d028640dffcc9203f8bd76/src/YakSwapCell.sol#L84) functions.


In particular, the `_swap` function might revert if decoding bad data. If that happens, tokens might be lost since the entire `_trySwap` [function](https://github.com/tesseract-protocol/smart-contracts/blob/940fc6650d1f48b3b3d028640dffcc9203f8bd76/src/Cell.sol#L140) would fail and the [assumption](https://github.com/tesseract-protocol/smart-contracts/blob/940fc6650d1f48b3b3d028640dffcc9203f8bd76/src/Cell.sol#L74) that `_swap` does not fail will be broken.


Consider managing eventual throws of the `abi.decode` call in the `_swap` function and thinking about the remaining examples on whether they need special attention and/or explicit documentation.


***Update:** Acknowledged, not resolved. The team stated:*



> *In the current version of Solidity, there is no way to implement a try/catch mechanism for `abi.decode`. While this is unfortunate for users who might accidentally pass incorrect data while attempting a swap, any resulting revert will be handled by the sending bridge. In such cases, the tokens will be sent to the fallback receiver. The route function is not called onchain by the protocol itself. The risk that users could potentially end up with tokens on an intermediate chain will be clearly stated in the frontend interface and documentation.*

---
