# Cluster -1118

**Rank:** #173  
**Count:** 72  

## Label
Missing or flawed fee validation leads to unauthorized fee setting, incorrect fee calculation, or denial-of-service, enabling economic manipulation or transaction failure.

## Cluster Information
- **Total Findings:** 72

## Examples

### Example 1

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
### Example 2

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
### Example 3

**Auto Label:** Missing or flawed fee validation leads to unauthorized fee setting, incorrect fee calculation, or denial-of-service, enabling economic manipulation or transaction failure.  

**Original Text Preview:**

**Description:** Currently the protocol allows users to over-pay:
```solidity
function _revertIfInsufficientFee() internal view {
    if (msg.value < _getFee()) revert SoulBoundToken__InsufficientFee();
}
```

Consider changing this to require the exact fee to prevent users from accidentally over-paying:
```solidity
function _revertIfIncorrectFee() internal view {
    if (msg.value != _getFee()) revert SoulBoundToken__IncorrectFee();
}
```

[Fat Finger](https://en.wikipedia.org/wiki/Fat-finger_error) errors have previously resulted in notorious unintended errors in financial markets; the protocol could choose to be defensive and help protect users from themselves.

**Evo:**
Fixed in commit [e3b2f74](https://github.com/contractlevel/sbt/commit/e3b2f74239601b2721118e11aaa92b42dbb502e9).

**Cyfrin:** Verified.

---
