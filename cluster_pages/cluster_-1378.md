# Cluster -1378

**Rank:** #243  
**Count:** 39  

## Label
Fee asymmetry and inconsistent fee application due to flawed conditional logic, leading to unfair user treatment, economic misrepresentation, and potential exploitation.

## Cluster Information
- **Total Findings:** 39

## Examples

### Example 1

**Auto Label:** Fee asymmetry and inconsistent fee application due to flawed conditional logic, leading to unfair user treatment, economic misrepresentation, and potential exploitation.  

**Original Text Preview:**

When converting wToken to rsToken, the fee is calculated as such:

```
 function calculateAmtAfterFee(uint256 wcomaiAmount)
    ...
    uint256 _bridgeFee = wcomaiAmount * bridgingFee / 1000;
    if(_bridgeFee < 1 * (10 ** decimals())) _bridgeFee = 1 * (10 ** decimals());
    uint256 amountAfterBridgingFee = wcomaiAmount - _bridgeFee;
    uint256 feeAmount = 0;
    if(stakingFee > 0) {
      feeAmount = (amountAfterBridgingFee * stakingFee) / 1000;
    }
    uint256 amountAfterTotalFees = amountAfterBridgingFee - feeAmount;
  }

```

The total amount deducts the bridging fee and then the new amount deducts the staking fee.

When unstaking, the bridging fee and unstaking fee are calculated on the total amount.

```
  function getWCOMAIByrsCOMAIAfterFee(uint256 rsCOMAIAmount)
    public
    view
    returns (uint256, uint256, uint256)
  {
    uint256 unstakingFeeAmt = rsCOMAIAmount * unstakingFee / 1000;
    uint256 bridgingFeeAmt = rsCOMAIAmount * bridgingFee / 1000;
    if(bridgingFeeAmt < 1 * (10 ** decimals())) bridgingFeeAmt = 1 * (10 ** decimals());
    uint256 unstakingAmt = rsCOMAIAmount - bridgingFeeAmt - unstakingFeeAmt;
    return (unstakingAmt, bridgingFeeAmt, unstakingFeeAmt);
  }
```

This means that the fee calculation is different when staking and unstaking. Users will pay a little less fee when staking given staking and unstaking percentage is the same.

For example, if both staking and unstaking are 1% and the bridge is 1%, a user with 1000 tokens will pay 10 tokens for unstaking and 10 tokens for the bridge fee, whereas the user will pay 10 tokens for the bridge fee and 9.9 tokens for staking.

Standardize the way the fee calculation is.

---
### Example 2

**Auto Label:** Misconfigured fee handling and improper administrative access lead to revenue loss, incorrect fee calculations, and unauthorized state modifications.  

**Original Text Preview:**

## Severity

**Impact:** Medium, because the flash loan functionality is affected

**Likelihood:** High, because it will revert every time а flash loan is used

## Description

`balanceBefore` is recorded before the flash loan amount being transferred. As a result, in line 714, `balanceAfter` need to be more than needed. Users have to pay extra with the same amount to use a flash loan.

```solidity
File: src\abstract\As4626.sol
696:     function flashLoanSimple() external nonReentrant {

705:         uint256 fee = exemptionList[msg.sender] ? 0 : amount.bp(fees.flash);
706:         uint256 toRepay = amount + fee;
707:
708:         uint256 balanceBefore = asset.balanceOf(address(this));
709:         totalLent += amount;
710:
711:         asset.safeTransferFrom(address(this), address(receiver), amount);
712:         receiver.executeOperation(address(asset), amount, fee, msg.sender, params);
713:
714:         if ((asset.balanceOf(address(this)) - balanceBefore) < toRepay)
715:             revert FlashLoanDefault(msg.sender, amount);

718:     }
```

## Recommendations

The `toRepay` variable could be dropped, or record the `balanceBefore` after `safeTransferFrom`:

```diff
File: src\abstract\As4626.sol
696:     function flashLoanSimple() external nonReentrant {

705:         uint256 fee = exemptionList[msg.sender] ? 0 : amount.bp(fees.flash);
- 706:         uint256 toRepay = amount + fee;
707:
708:         uint256 balanceBefore = asset.balanceOf(address(this));
709:         totalLent += amount;
710:
711:         asset.safeTransferFrom(address(this), address(receiver), amount);
712:         receiver.executeOperation(address(asset), amount, fee, msg.sender, params);
713:
- 714:         if ((asset.balanceOf(address(this)) - balanceBefore) < toRepay)
+ 714:         if ((asset.balanceOf(address(this)) - balanceBefore) < fee)
715:             revert FlashLoanDefault(msg.sender, amount);

718:     }

```

---
### Example 3

**Auto Label:** Misconfigured fee handling and improper administrative access lead to revenue loss, incorrect fee calculations, and unauthorized state modifications.  

**Original Text Preview:**

The `updateDexConfig()` function in `PoolAdminFacet` sets a single fee value per DEX router address:

```solidity
function updateDexConfig(address router, uint24 fee, bool isValid)
    external
    onlyRole(AccessControlStorage.DEFAULT_ADMIN_ROLE)
{
    PoolStorage.Dex storage dex = PoolStorage.layout().setUp.dex;
    dex.valid[router] = isValid;
@>  dex.fee[router] = fee;
    emit DexUpdated(router, isValid, fee);
}
```

However, this structure fails to accommodate DEXes like Uniswap V3, which supports multiple pools per token pair with different fees (e.g., 500, 3000, 10000). Using a single fee for all swaps on a given router can result in a revert or higher slippage in `_swapAssetToAsset` function.

Recommendations:

Update the design to allow pair-specific fee configuration.

---
