# Cluster -1296

**Rank:** #216  
**Count:** 50  

## Label
**Improper fee handling due to flawed logic, state inconsistency, or incorrect assumptions about fee application, leading to double charges, loss of funds, or fee evasion.**

## Cluster Information
- **Total Findings:** 50

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

**Auto Label:** Misconfigured fee handling and improper administrative access lead to revenue loss, incorrect fee calculations, and unauthorized state modifications.  

**Original Text Preview:**

[/contracts/pool-manager/src/manager/commands.rs# L81](https://github.com/code-423n4/2024-11-mantra-dex/blob/26714ea59dab7ecfafca9db1138d60adcf513588/contracts/pool-manager/src/manager/commands.rs# L81)

### Impact

For every swap operation, the protocol is entitled to a get fees from the tokens out, the sponsor has confirmed that fees should be taken for every swap. However, the current implementation incorrectly gives the `protocol_fee` control to pool creators when creating pool:
```

pub fn create_pool(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    asset_denoms: Vec<String>,
    asset_decimals: Vec<u8>,
    pool_fees: PoolFee,
    pool_type: PoolType,
    pool_identifier: Option<String>,
) -> Result<Response, ContractError> {
    // --SNIP
    POOLS.save(
        deps.storage,
        &identifier,
        &PoolInfo {
            pool_identifier: identifier.clone(),
            asset_denoms,
            pool_type: pool_type.clone(),
            lp_denom: lp_asset.clone(),
            asset_decimals,
            pool_fees,
            assets,
        },
    )?;
}
```

Pool creators can specify 0 fees for `pool_fees.protocol_fee`, causing the protocol to lose swap fees they are entitled to.

### Recommended mitigation steps

Consider moving the `protocol_fee` to the contract’s config, rather than being controlled by pool creators.

**jvr0x (MANTRA) confirmed**

**[3docSec (judge) commented](https://code4rena.com/audits/2024-11-mantra-dex/submissions/F-25?commentParent=TkuLQA35hAR):**

> Looks valid, probably Low though, as there is no impact on users.

**[DadeKuma (warden) commented](https://code4rena.com/audits/2024-11-mantra-dex/submissions/F-25?commentParent=TkuLQA35hAR&commentChild=KYDMgxMtCw4):**

> This should be Low/info at most because `pool_fees.protocol_fee` is a “frontend” fee (i.e., a fee charged for using the user interface that facilitates interaction with these contracts). This is common in many protocols.
>
> The main fees, which are also implemented correctly (and fetched from the config), are the pool creation fee and the token factory fee.

**[Abdessamed (warden) commented](https://code4rena.com/audits/2024-11-mantra-dex/submissions/F-25?commentParent=TkuLQA35hAR&commentChild=xqoT9F2ReFA):**

> @DadeKuma - the fees you are talking about are the **pool creation fees**, which is implemented correctly and the report does not speak about these fees at all.
>
> What the report is highlighting is that the **swap fees** are given control to the pool creators mistakenly rather than being imposed by the protocol’s configuration whereby pool creators can simply put zero fees for the protocol team to prevent them from taking swap fees for whatever reason. The sponsor intends to take fees from every swap operation to collect revenue, which is not implemented.
>
> @3docSec - there is no impact for pool creators but at **the expense of protocol losses**. The protocol team intends to collect revenue from swap fees (like Uniswap and other AMMs do), and not receiving those fees is a clear loss for the protocol team, and thus, this report is eligible for Medium severity.

**[3docSec (judge) commented](https://code4rena.com/audits/2024-11-mantra-dex/submissions/F-25?commentParent=TkuLQA35hAR&commentChild=kwp3zBJXt45):**

> @Abdessamed - I agree. I do not think this is a likely attack vector because the protocol can:
>
> * Force through the UI a proper value if creators create pools through the UI.
> * Not show in the UI pools that don’t have an adequate protocol fee for swap users.
>
> We are a bit borderline here because of likelihood, but Medium seems justified to me.

---

---
