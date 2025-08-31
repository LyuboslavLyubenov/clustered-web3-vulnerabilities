# Cluster -1200

**Rank:** #108  
**Count:** 155  

## Label
Failure to account for transaction fees in token transfers leads to incorrect liquidity calculations, fund loss, and unintended fee charges, enabling users to gain unfair advantages or lose protocol value.

## Cluster Information
- **Total Findings:** 155

## Examples

### Example 1

**Auto Label:** Failure to account for transfer fees or dynamic token balances leads to incorrect state updates, fund loss, or overstatements of user balances due to lack of on-chain validation and atomicity.  

**Original Text Preview:**

The `RivusTAO::wrap` function checks if the `wTAO` amount exceeds the `minStakingAmt` before fees are subtracted:

```solidity
File: RivusTAO.sol
887:   function wrap(uint256 wtaoAmount) public nonReentrant checkPaused returns (uint256) {
...
...
918:     // Ensure that at least 0.125 TAO is being bridged
919:     // based on the smart contract
920:     require(wtaoAmount > minStakingAmt, "Does not meet minimum staking amount");
921:
922:
923:     // Ensure that the wrap amount after free is more than 0
924:     (uint256 wrapAmountAfterFee, uint256 feeAmt) = calculateAmtAfterFee(wtaoAmount);
925:
926:     uint256 rsTAOAmount = getRsTAObyWTAO(wrapAmountAfterFee);
...
...
939:   }
```

This approach is flawed as it should ensure that the net amount after fees, which is actually being staked, meets the `minStakingAmt`. Performing the check prior to fee deductions can lead to scenarios where the actual staked amount is less than the intended minimum due to the fees subtracted afterward.

Adjust the validation process to check `minStakingAmt` after the fees have been calculated and deducted from the `wTAO` amount. This ensures that the amount effectively being staked still meets the minimum requirements stipulated by the protocol, preventing users from staking less than the minimum due to fees.

```diff
  function wrap(uint256 wtaoAmount) public nonReentrant checkPaused returns (uint256) {
    ...
    ...

--  // Ensure that at least 0.125 TAO is being bridged
--  // based on the smart contract
--  require(wtaoAmount > minStakingAmt, "Does not meet minimum staking amount");

    // Ensure that the wrap amount after free is more than 0
    (uint256 wrapAmountAfterFee, uint256 feeAmt) = calculateAmtAfterFee(wtaoAmount);

++  // Ensure that at least 0.125 TAO is being bridged
++  // based on the smart contract
++  require(wrapAmountAfterFee > minStakingAmt, "Does not meet minimum staking amount");

    uint256 rsTAOAmount = getRsTAObyWTAO(wrapAmountAfterFee);
    ...
    ...
  }
```

---
### Example 2

**Auto Label:** Failure to account for transfer fees or dynamic token balances leads to incorrect state updates, fund loss, or overstatements of user balances due to lack of on-chain validation and atomicity.  

**Original Text Preview:**

## Severity

Medium Risk

## Description

The `MultipliBridger` contract's deposit mechanism fails to properly account for fee-on-transfer (FoT) tokens. The vulnerability manifests in the `deposit` function, which invokes `TransferHelper.safeTransferFrom(token, msg.sender, address(this), amount)` to transfer tokens from the user to the contract. This implementation assumes the amount specified will exactly match the tokens received by the contract, which is incorrect for FoT tokens that deduct a transfer fee.

The contract then emits the `BridgedDeposit event` with the full pre-fee amount as the deposited value. Off-chain systems, including the sequencer and bridging back-end that monitor these events, will interpret this logged amount as the actual value received by the contract. This creates a discrepancy where users receive receipt tokens (`xTokens on L2`) based on the pre-fee amount while the contract holds less value than recorded.

## Location of Affected Code

File: [src/MultipliBridger.sol#L125](https://github.com/multipli-libs/multipli-bridger/blob/c5b4869fcf7349c9ce9b67ca26f6ae45cc7babce/src/MultipliBridger.sol#L125)

```solidity
function deposit(address token, uint256 amount) external {
  TransferHelper.safeTransferFrom(
    token,
    msg.sender,
    address(this),
    amount
  );
  emit BridgedDeposit(msg.sender, token, amount);
}
```

## Impact

When FoT tokens are deposited, the protocol will credit users with receipt tokens based on the pre-fee amount while holding less actual token value in the contract. This creates an accounting imbalance that could lead to liquidity shortfalls when processing withdrawals.

## Recommendation

The contract should implement FoT token handling by comparing the balance before and after transfers. For the `deposit` function, first record the contract's token balance, then perform the transfer, then verify the new balance to determine the actual received amount.

## Team Response

Fixed.

---
### Example 3

**Auto Label:** Failure to account for transfer fees or dynamic token balances leads to incorrect state updates, fund loss, or overstatements of user balances due to lack of on-chain validation and atomicity.  

**Original Text Preview:**

## Severity: Low Risk

## Context
(No context files were provided by the reviewers)

## Description
In both sync and async deposit flows — as seen in **Provisioner.sol#L158-L191** and **Provisioner.sol#L436-L476** — the protocol assumes that the amount of tokens transferred via `transferFrom` or `safeTransferFrom` will match the intended deposit amount exactly.

This logic is not compatible with:
- Fee-on-transfer tokens (e.g., tokens that deduct a percentage on each transfer)
- Rebasing tokens that adjust balances after deposit
- Deflationary or redirecting tokens that reduce the net amount received
- Tokens with hook-based transfer logic or non-standard `transferFrom` behaviors

Since the amount transferred is not validated via a pre/post balance delta or adjusted in response to what the vault actually receives, any mismatch may result in silent value loss or incorrect vault accounting. The user may receive fewer shares than expected, or vault totals may drift.

## Recommendation
It may be helpful to document clearly that only standard ERC20 tokens are supported and to explicitly warn against depositing tokens with fee-on-transfer or rebasing behavior.

In the longer term, supporting these token types would require implementing post-transfer balance checks (e.g., comparing `balanceBefore` and `balanceAfter`) or deposit value quoting based on actual received amounts — but this would increase gas cost and complexity.

## Aera
Acknowledged. Will not fix.

## Spearbit
Acknowledged by Aera team.

---
