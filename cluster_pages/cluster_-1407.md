# Cluster -1407

**Rank:** #174  
**Count:** 71  

## Label
Attackers manipulate utilization or collateralization ratios via flash loans or state manipulation, bypassing safeguards to achieve inflated borrowing, drain liquidity, or trigger system failures without proper validation or locking mechanisms.

## Cluster Information
- **Total Findings:** 71

## Examples

### Example 1

**Auto Label:** Flash loan vulnerabilities enabling unauthorized asset manipulation, fee bypass, or supply overdraw through flawed validation, precision errors, or improper recipient checks.  

**Original Text Preview:**

**Impact**
The code for the Balancer Vault Flashloan is as follows:

https://etherscan.io/address/0xba12222222228d8ba445958a75a0704d566bf2c8#code

```solidity
   function flashLoan(
        IFlashLoanRecipient recipient,
        IERC20[] memory tokens,
        uint256[] memory amounts,
        bytes memory userData
    ) external override nonReentrant whenNotPaused {
        InputHelpers.ensureInputLengthMatch(tokens.length, amounts.length);

        uint256[] memory feeAmounts = new uint256[](tokens.length);
        uint256[] memory preLoanBalances = new uint256[](tokens.length);

        // Used to ensure `tokens` is sorted in ascending order, which ensures token uniqueness.
        IERC20 previousToken = IERC20(0);

        for (uint256 i = 0; i < tokens.length; ++i) {
            IERC20 token = tokens[i];
            uint256 amount = amounts[i];

            _require(token > previousToken, token == IERC20(0) ? Errors.ZERO_TOKEN : Errors.UNSORTED_TOKENS);
            previousToken = token;

            preLoanBalances[i] = token.balanceOf(address(this));
            feeAmounts[i] = _calculateFlashLoanFeeAmount(amount);

            _require(preLoanBalances[i] >= amount, Errors.INSUFFICIENT_FLASH_LOAN_BALANCE);
            token.safeTransfer(address(recipient), amount);
        }

        recipient.receiveFlashLoan(tokens, amounts, feeAmounts, userData);
```

Meaning anyone can call this and set the `BalancerFlashloan` as the recipient

This only check done in the Zappers looks as follows:

https://github.com/liquity/bold/blob/57c2f13fe78f914cab3d9736f73b4e0ffc58e022/contracts/src/Zappers/LeverageWETHZapper.sol#L165-L168

```solidity
    function receiveFlashLoanOnLeverDownTrove(LeverDownTroveParams calldata _params, uint256 _effectiveFlashLoanAmount)
        external
    {
        require(msg.sender == address(flashLoanProvider), "LZ: Caller not FlashLoan provider");
```

Which means that any arbitrary flashloan can be made to trigger the callbacks specified in `receiveFlashLoan`

This opens up to the following:

**We can do the FL on behalf of other people and cause them to unwind or take on more debt**

Since balancer removed the flashloan fee


This can be abused to alter a user position:
- Force them to take leverage or to unwind their position

And can also be used to steal funds from the user by:
- Imbalancing swap pool
- Having user alter their position and take a loss / more debt than intended

The loss will be taken by the victim in the form of a "imbalanced" leverage

Whereas the attacker will skim the sandwhiched funds to themselves



**Proof Of Code**

- Call `flashLoanProvider.makeFlashLoan` with malicious inputs
OR
- Call Vault.flashloan with malicious inputs

<img width="886" alt="Screenshot 2024-09-03 at 15 23 41" src="https://github.com/user-attachments/assets/0cc97368-bd41-427c-8b85-2b50ba9bb151">

<img width="1270" alt="Screenshot 2024-09-03 at 15 23 52" src="https://github.com/user-attachments/assets/0e940408-2fde-4ece-91d7-cb8d720b0850">

<img width="1178" alt="Screenshot 2024-09-03 at 15 24 01" src="https://github.com/user-attachments/assets/7ac7a8e0-3e93-4d86-94f5-19099a9f7f95">


<img width="1334" alt="Screenshot 2024-09-03 at 15 24 10" src="https://github.com/user-attachments/assets/824207da-8c9c-400b-a0f9-8bc1b3ae6595">


**Mitigation**

Enforce that all calls are started by the Zaps (trusted contracts)

You could generate a transient id for the operation, and verify that the only accepted callback is the trusted one

**Pseudocode**

```solidity
    function receiveFlashLoanOnOpenLeveragedTrove(
        OpenLeveragedTroveParams calldata _params,
        uint256 _effectiveFlashLoanAmount
    ) external {
        require(msg.sender == address(flashLoanProvider), "LZ: Caller not FlashLoan provider");
        require(decodeParams(_params) == currentOperationId, "callback was started by this contract");
        currentOperationId = UNSET_OPERATION_ID; // bytes32(0)
```

---

---
### Example 2

**Auto Label:** Flash loan vulnerabilities enabling unauthorized asset manipulation, fee bypass, or supply overdraw through flawed validation, precision errors, or improper recipient checks.  

**Original Text Preview:**

**Impact**
These 3 lines can be VERY dangerous

https://github.com/GalloDaSballo/quill-review/blob/8d6e4c8ed0759cea1ff0376db9fd55db864cd7e8/contracts/src/Zappers/Modules/FlashLoans/BalancerFlashLoan.sol#L46-L52

```solidity
        // This will be used by the callback below no
        receiver = IFlashLoanReceiver(msg.sender);

        vault.flashLoan(this, tokens, amounts, userData);

        // Reset receiver
        receiver = IFlashLoanReceiver(address(0));
```

The reason why this code is safe for BOLD is becasue `vault` has a Reentrancy guard

In lack of that guard many projects can get exploited

**Theoretical Proof Of Code**

https://gist.github.com/GalloDaSballo/a4dd8c2b77a64d602983152d621f55c3

**Theoretical Mitigation**

- Validate AAVE initiator
- Consume the receiver

```solidity
    function receiveFlashLoan(
        IERC20[] calldata tokens,
        uint256[] calldata amounts,
        uint256[] calldata feeAmounts,
        bytes calldata userData
    ) external override {
        require(msg.sender == address(vault), "Caller is not Vault");
        require(address(receiver) != address(0), "Flash loan not properly initiated");

        // NOTE: Validate initiator (if available) (e.g. AAVE)
        // NOTE: Why not consume receiver?
        IFlashLoanReceiver public cachedReceiver = receiver;
        receiver = IFlashLoanReceiver(address(0));
```

---
### Example 3

**Auto Label:** Flash loan vulnerabilities enabling unauthorized asset manipulation, fee bypass, or supply overdraw through flawed validation, precision errors, or improper recipient checks.  

**Original Text Preview:**

## Vulnerability Report: Double Upscaling in pay_flashloan

## Description

The vulnerability arises from double upscaling during the repayment process in `pay_flashloan` when handling meta-stable pools. Specifically, `pay_flashloan` upscales `balance_after_flashloan` twice. When handling meta-stable pools, the funds are multiplied by their value derived from an oracle. As a result, the post-repayment invariant computation utilizes an incorrectly scaled value.

## Code Snippet

> _ thalaswap_v2/sources/pool.move rust

```rust
public fun pay_flashloan(assets: vector<FungibleAsset>, loan: Flashloan) acquires PauseFlag,
,→ Pool, MetaStablePool, StablePool, ThalaSwap, WeightedPool {
    [...]
    if (pool_is_metastable(pool_obj)) {
        borrow_amounts = upscale_metastable_amounts(pool_obj, borrow_amounts);
        balances = upscale_metastable_amounts(pool_obj, pool_balances(pool_obj));
    };
    [...]
    while (i < len) {
        let repay_sub_fees = *vector::borrow(&repay_amounts, i) - *vector::borrow(&fee_amounts,
        ,→ i);
        let balance_after_flashloan = *vector::borrow(&balances, i) + repay_sub_fees;
        vector::push_back(&mut balances_after_flashloan, balance_after_flashloan);
        i = i + 1;
    };
    [...]
    if (pool_is_metastable(pool_obj)) {
        balances_after_flashloan = upscale_metastable_amounts(pool_obj,
        ,→ balances_after_flashloan);
    };
    [...]
}
```

Consequently, this creates a discrepancy where the post-repayment invariant (`curr_invariant`) appears larger than the pre-repayment invariant (`prev_invariant`), even if no real repayment has occurred. This incorrect invariant validation allows the flashloan repayment to proceed without properly verifying the adequacy of the repayment. Thus, the pool may accept a repayment that is less than the borrowed amount, resulting in a loss of funds.

## Remediation

Eliminate the double upscaling to prevent inflation of the `curr_invariant`.

## Patch

Fixed in commit `19dc5f1`.

---
