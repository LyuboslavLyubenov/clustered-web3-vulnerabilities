# Cluster -1288

**Rank:** #53  
**Count:** 325  

## Label
Precision loss and logic flaws in reward calculations due to improper handling of fixed-point arithmetic, incorrect loop bounds, and inconsistent state updates, leading to systematic underpayment, denial of service, and wealth leakage.

## Cluster Information
- **Total Findings:** 325

## Examples

### Example 1

**Auto Label:** Missing input or state validation leads to exploitable fee or parameter manipulation, enabling incorrect logic, overflow, or unfair economic outcomes.  

**Original Text Preview:**

In the `updateFeeToken`, `updateFeeAmount`, and `updateFeeTo` functions, the fee config is not checked to ensure it has been properly set beforehand. As a result, an editor could mistakenly update the fee config for a chain ID where the fee config has not been set.

Example of impact: If the editor updates the fee token, the fee config could not be set for that chain ID.

---
### Example 2

**Auto Label:** Inconsistent fee handling and data validation across critical functions lead to inaccurate cost estimates, financial loss, and protocol dysfunction due to flawed logic, missing checks, or misaligned state assumptions.  

**Original Text Preview:**

## Severity

**Impact:** Low, because wrong accounting of fee

**Likelihood:** High, because it will happen in each call to deposit/mint

## Description

In `_deposit()` function code calculates fee like this:

```solidity
        // slice the fee from the amount (gas optimized)
        if (!exemptionList[_receiver])
            claimableAssetFees += _amount.revBp(fees.entry);
```

And the `revBp()` logic is:

```solidity
    function revBp(
        uint256 amount,
        uint256 basisPoints
    ) internal pure returns (uint256) {
        return mulDiv(amount, basisPoints, BP_BASIS - basisPoints);
    }
```

As the `amount` in the deposit is not sliced and it is `deposit + fee` so the calculation for fee is wrong.

## Recommendations

Fee calculation should be:

```solidity
             claimableAssetFees  += _amount * fees.entry / (BP_BASIS  + fees.entry)
```

---
### Example 3

**Auto Label:** Inconsistent fee handling and data validation across critical functions lead to inaccurate cost estimates, financial loss, and protocol dysfunction due to flawed logic, missing checks, or misaligned state assumptions.  

**Original Text Preview:**

## Severity

**Impact:** High, because it can cause double spending and disturb the pool calculations

**Likelihood:** Medium, because exception recipients are set by admin for common addresses

## Description

When users want to deposit their tokens, code calculates fee in `previewDeposit()` and `previewMint()` and add/subtract it from the amount/share. As you can see code checks entry fee based on `msg.sender`:

```solidity
    function previewDeposit(
        uint256 _amount
    ) public view returns (uint256 shares) {
        return convertToShares(_amount).subBp(exemptionList[msg.sender] ? 0 : fees.entry);
    }

    function previewMint(uint256 _shares) public view returns (uint256) {
        return convertToAssets(_shares).addBp(exemptionList[msg.sender] ? 0 : fees.entry);
    }
```

In `_deposit()`, code wants to account the fee and keep track of it, it perform this action:

```solidity
        if (!exemptionList[_receiver])
            claimableAssetFees += _amount.revBp(fees.entry);
```

As you can see it uses `_receiver` variable which is controllable by caller.

So attacker with two addresses: RECV1 address and has set as `exemptionList[]` can ADDERSS1 which doesn't set as `exemptionList[]` can call mint and deposit with ADDRESS1 and set recipient as RECV1 . In preview functions code doesn't add the fee for user deposit(or subtract it from shares) and code would assume user didn't pay any fee, but in the `_deposit()` function code would check RECV1 address and would add calculated fee to accumulated fee. So in the end user didn't paid the fee but code added fee and double spending would happen.

Attacker can use another scenarios to perform this issue too. (code calculates fee and caller pays it but code doesn't add it to `claimableAssetFees `)

---

When users want to withdraw their tokens, Code charges fee and it's done in preview functions by adding/subtracting fee from amount/share. As you can see code checks exit fee based on `exemptionList[]` and uses `msg.sender` as target:

```solidity
    function previewWithdraw(uint256 _assets) public view returns (uint256) {
        return convertToShares(_assets).addBp(exemptionList[msg.sender] ? 0 : fees.exit);
    }

    function previewRedeem(uint256 _shares) public view returns (uint256) {
        return convertToAssets(_shares).subBp(exemptionList[msg.sender] ? 0 : fees.exit);
    }
```

But in `_withdraw()` function when code wants to calculates fee and accumulated it, it uses `_owner`:

```solidity
        if (!exemptionList[_owner])
            claimableAssetFees += _amount.revBp(fees.exit);
```

`owner` can be different that caller(`msg.sender`) and code checks that caller have approval over the `owner`'s funds.
So attacker can exploit this with two of his address: OWNER1 which has set as `exemptionList[]` and OPERATOR1 which doesn't set as `exemptionList[]`. If attacker give approval of OWNER1 tokens to OPERATOR1 and calls `withdraw(OWNER1)` with OPERATOR1 address then double spend would happen. While code returns funds fully with no charged fee it would also add fee to `claimableAssetFees`.

This can be exploited in other scenarios too. In general this inconsistency would cause accountant errors.

## Recommendations

Calculate the fee based on `msg.sender` in the `_deposit()` function.
In `_withdraw()` function calculate fee based on `msg.sender` and finally fix the `preview` functions.

---
