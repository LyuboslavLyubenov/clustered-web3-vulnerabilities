# Cluster -1353

**Rank:** #114  
**Count:** 149  

## Label
Inconsistent fee handling and data validation across critical functions lead to inaccurate cost estimates, financial loss, and protocol dysfunction due to flawed logic, missing checks, or misaligned state assumptions.

## Cluster Information
- **Total Findings:** 149

## Examples

### Example 1

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
### Example 2

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
### Example 3

**Auto Label:** Inconsistent fee handling and data validation across critical functions lead to inaccurate cost estimates, financial loss, and protocol dysfunction due to flawed logic, missing checks, or misaligned state assumptions.  

**Original Text Preview:**

## Severity

**Impact:** Medium, fee will be a little higher/lower

**Likelihood:** High, because it happens in every call to mint and deposit functions

## Description

When users calls `mint(shares)` code calls `_deposit(previewMint(_shares), _shares` and `previewMint(shares) = convertToAssets(shares).addBp()`.
When users calls `deposit(amount)` code calls `_deposit(_amount, previewDeposit(_amount)` and `previewDeposit(amount) = convertToShares(amount).subBp`.

Let's assume that price is 1:1 and fee is 10% and check the both case:

1. If user wants to mint 100 share then he would call `mint(100)` and code would calculate `amount = previewMint(100) = convertToAssets(100).addBp(10%) = 110`. So in the end user would pay 110 asset and receive 100 shares and 10 asset will be fee.
2. If users wants to deposit 110 asset then he would call `deposit(110)` and code would calculate `share = previewDeposit(110) = convertToShare(110).subBp(10%) = 99`. So in the end user would pay 11 asset and receive 99 share and 11 asset will be fee.

As you can see the `deposit()` call overcharge the user. The reason is that code calculates fee based on user-specified amount by using `subBp()` but user-specified amount is supposed to be `amount + fee` so the calculation for fee should be `.... * base / (base +fee)`.

---

When users call `redeem(shares)` code calls `_withdraw(previewRedeem(_shares), _shares)` and `previewRdeem(shares) = convertToAssets(_shares).subBp()`.

When users call `withdraw()` code calls `_withdraw(_amount, previewWithdraw(_amount))` and `previewWithdraw(_amount) = convertToShares(_assets).addBp()`

Let's assume that asset to share price is 1:1 and fee is 10% and check both case:

1. If user wants to redeem 110 shares then he would call `redeem(110)` and code would calculate `amount = previewRdeem(110) = convertToAssets(110).subBp(10%) = 99`. So in the end user would burn 110 shares and receive 99 asset and 11 asset will be fee.
2. If user wants to withdraw 100 asset then he would call `withdraw(100)` and code would calculates `shares = previewWithdraw(100) = convertToShares(100).addBp(10%) = 110`. So in the end userwould burn 110 shares and receive 100 asset and 10 asset will be fee.

So as you can see `redeem()` overcharges users. The reason is that code calculates fee based on user provided share with `subBp()` but the provided amount is total amount (burnAmount + fee) and calculation should be `..... * base / (base + fee)`

## Recommendations

Calculate the fee for `deposit()` with `convertToShare(amount) * base / (base +fee)`.
Calculate the fee for `previewRedeem()` with `convertToAssets(shares) * base / (base + fee)`

---
