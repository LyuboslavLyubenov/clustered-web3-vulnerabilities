# Cluster -1230

**Rank:** #90  
**Count:** 193  

## Label
Inconsistent fee handling and data validation across critical functions lead to inaccurate cost estimates, financial loss, and protocol dysfunction due to flawed logic, missing checks, or misaligned state assumptions.

## Cluster Information
- **Total Findings:** 193

## Examples

### Example 1

**Auto Label:** Failure to properly persist or validate fees in storage or logic, resulting in irreversible user fund loss, incorrect fee accumulation, or ineffective fee control.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

Inside `CLGauge.claimFees`, it will call `pool.collectFees` and if returned `claimed0` and `claimed1` is greater than 0, it will consider it as fees.

```solidity
    function _claimFees() internal returns (uint claimed0, uint claimed1) {
        if (!isForPair) return (0, 0);
        (claimed0, claimed1) = pool.collectFees();
>>>     if (claimed0 > 0 || claimed1 > 0) {
            uint _fees0 = fees0 + claimed0;
            uint _fees1 = fees1 + claimed1;

            if (
                _fees0 > IBribe(internal_bribe).left(token0) &&
                _fees0 / ProtocolTimeLibrary.WEEK > 0
            ) {
                fees0 = 0;
                _safeApprove(token0, internal_bribe, _fees0);
                IBribe(internal_bribe).notifyRewardAmount(token0, _fees0);
            } else {
                fees0 = _fees0;
            }
            if (
                _fees1 > IBribe(internal_bribe).left(token1) &&
                _fees1 / ProtocolTimeLibrary.WEEK > 0
            ) {
                fees1 = 0;
                _safeApprove(token1, internal_bribe, _fees1);
                IBribe(internal_bribe).notifyRewardAmount(token1, _fees1);
            } else {
                fees1 = _fees1;
            }

            emit ClaimFees(msg.sender, claimed0, claimed1);
        }
    }
```

However, inside `pool`, it will set fees amount to 1 if it is empty.

```solidity
    function collectFees()
        external
        override
        lock
        onlyGauge
        returns (uint128 amount0, uint128 amount1)
    {
        amount0 = gaugeFees.token0;
        amount1 = gaugeFees.token1;
        if (amount0 > 1) {
            // ensure that the slot is not cleared, for gas savings
>>>         gaugeFees.token0 = 1;
            TransferHelper.safeTransfer(token0, msg.sender, --amount0);
        }
        if (amount1 > 1) {
            // ensure that the slot is not cleared, for gas savings
>>>         gaugeFees.token1 = 1;
            TransferHelper.safeTransfer(token1, msg.sender, --amount1);
        }

        emit CollectFees(msg.sender, amount0, amount1);
    }
```

This means if currently fees are empty, `CLGauge` will incorrectly add 1 fee to the `fees0` and `fees1`, which will eventually a cause revert due to lack of balance.

## Recommendations

Adjust the following line inside `CLPool`.

```diff
    function collectFees() external override lock onlyGauge returns (uint128 amount0, uint128 amount1) {
        amount0 = gaugeFees.token0;
        amount1 = gaugeFees.token1;
        if (amount0 > 1) {
            // ensure that the slot is not cleared, for gas savings
            gaugeFees.token0 = 1;
            TransferHelper.safeTransfer(token0, msg.sender, --amount0);
-        }
+        } else {
+           --amount0;
+      }
        if (amount1 > 1) {
            // ensure that the slot is not cleared, for gas savings
            gaugeFees.token1 = 1;
            TransferHelper.safeTransfer(token1, msg.sender, --amount1);
-        }
+        } else {
+           --amount1;
+      }

        emit CollectFees(msg.sender, amount0, amount1);
    }
```

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
