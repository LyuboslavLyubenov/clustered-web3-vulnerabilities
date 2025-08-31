# Cluster -1194

**Rank:** #316  
**Count:** 22  

## Label
Inaccurate decimal handling in share and asset calculations leads to flawed liquidity, pricing, and conversion mechanics, enabling attacks or capital misallocation through arithmetic errors and improper normalization.

## Cluster Information
- **Total Findings:** 22

## Examples

### Example 1

**Auto Label:** Inaccurate decimal handling in share and asset calculations leads to flawed liquidity, pricing, and conversion mechanics, enabling attacks or capital misallocation through arithmetic errors and improper normalization.  

**Original Text Preview:**

**Description:** When the vault share prices are computed in `BunniHookLogic::_shouldSurgeFromVaults`, the logic assumes that the vault share token decimals will be equal to the decimals of the underlying asset:

```solidity
// compute current share prices
uint120 sharePrice0 =
    bunniState.reserve0 == 0 ? 0 : reserveBalance0.divWadUp(bunniState.reserve0).toUint120();
uint120 sharePrice1 =
    bunniState.reserve1 == 0 ? 0 : reserveBalance1.divWadUp(bunniState.reserve1).toUint120();
// compare with share prices at last swap to see if we need to apply the surge fee
// surge fee is applied if the share price has increased by more than 1 / vaultSurgeThreshold
shouldSurge = prevSharePrices.initialized
    && (
        dist(sharePrice0, prevSharePrices.sharePrice0)
            > prevSharePrices.sharePrice0 / hookParams.vaultSurgeThreshold0
            || dist(sharePrice1, prevSharePrices.sharePrice1)
                > prevSharePrices.sharePrice1 / hookParams.vaultSurgeThreshold1
    );
```

Here, `reserveBalance0/1` is in the decimals of the underlying asset, whereas `bunniState.reserve0/1` is in the decimals of the vault share token. As a result, the computed `sharePrice0/1` could be significantly more/less than the expected 18 decimals.

While the ERC-4626 specification strongly recommends that the vault share token decimals mirror those of the underlying asset, this is not always the case. For example, this [Morpho vault](https://etherscan.io/address/0xd508f85f1511aaec63434e26aeb6d10be0188dc7) has an 18-decimal share token whereas the underlying WBTC has 8 decimals. Such a vault strictly conforming to the standard would break the assumption that share prices are always in `WAD` precision, but rather 8 corresponding to the underlying, and could result in a surge being considered necessary even if this shouldn't have been the case.

Considering a `vaultSurgeThreshold` of `1e3` as specified in the tests, the logic will trigger a surge when the absolute share price difference is greater than 0.1%. In the above case where share price is in 8 decimals, assuming that the share price is greater than 1 would mean that the threshold could be specified as at most `0.000001%`. This is not a problem when the maximum possible threshold of `type(uint16).max` is specified, since this corresponds to approximately `0.0015%`, but if the vault has negative yield then this can make the share price drop below 1 and the possible precision even further with it, amplified further still for larger differences between vault/asset decimals that result in even lower-precision share prices. In the worst case, while quite unlikely, this division rounds down to zero and a surge is executed when it is not actually needed, since any absolute change in share price will be greater than zero. The threshold is immutable, so it would not be trivial to avoid such a situation.

Additionally, this edge case results in the possibility for vault share prices to overflow `uint120` when the share token decimals are smaller than those of the underlying asset. For example, a 6-decimal share token for an 18-decimal underlying asset would overflow when a single share is worth more than `1_329_388` assets.

**Impact:** Surging when not needed may cause existing rebalance orders to be cleared when this is not actually desired, as well as computing a larger dynamic/surge fee than should be required which could prevent users from swapping. Overflowing share prices would result in DoS for swaps.

**Recommended Mitigation:** Explicitly handle the vault share token and underlying asset decimals to ensure that share prices are always in the expected 18-decimal precision.

**Bacon Labs:** Fixed in [PR \#100](https://github.com/timeless-fi/bunni-v2/pull/100).

**Cyfrin:** Verified, the token/vault values are now explicitly scaled during share price calculation in `_shouldSurgeFromVaults()`.

---
### Example 2

**Auto Label:** Inaccurate decimal handling in share and asset calculations leads to flawed liquidity, pricing, and conversion mechanics, enabling attacks or capital misallocation through arithmetic errors and improper normalization.  

**Original Text Preview:**

## Summary

In the current implementation, the calculation of `decimalOffset` is dynamic and based on the difference between `SYSTEM_DECIMALS` (18) and the decimals of the `indexToken`:

[Source Code Reference](https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/market-making/branches/VaultRouterBranch.sol#L173)

```solidity
uint8 decimalOffset = Constants.SYSTEM_DECIMALS - IERC20Metadata(vault.indexToken).decimals();
```

This offset is used in the formula:

```solidity
IERC4626(vault.indexToken).totalSupply() + 10 ** decimalOffset
```

The purpose of adding `10 ** decimalOffset` is to introduce a **virtual share adjustment** to mitigate donation attacks.

[OpenZeppelin ERC-4626 Documentation](https://docs.openzeppelin.com/contracts/4.x/erc4626)

## Issues with Dynamic Decimal Offset

## 1. **Unpredictability**
   - The value of `decimalOffset` depends on `indexToken.decimals()`, leading to inconsistent behavior across different tokens.
   - If `indexToken.decimals()` is 18, `decimalOffset` is `0`, effectively bypassing the intended protection.
   - If `indexToken.decimals()` is 6, `decimalOffset` is `12`, which may be overly restrictive.

### 2. **Dependency on `IERC20Metadata(vault.indexToken).decimals()`**
   - The `decimals()` function returns the sum of the asset token’s decimals and `decimalsOffset`:



   ```solidity
   function decimals() public view virtual override(IERC20Metadata, ERC20Upgradeable) returns (uint8) {
       ERC4626Storage storage $ = _getERC4626Storage();
       return $._underlyingDecimals + _decimalsOffset();
   }
   ```
 - `decimalsOffset` is set by the deployer of the ZLP vault.

   https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/zlp/ZlpVault.sol#L74

   ```solidity
   zlpVaultStorage.decimalsOffset = decimalsOffset;
   ```

  
   - If an 18-decimal asset is used, `decimalsOffset` must be set to zero to ensure that `indexToken.decimals()` remains 18.

   - This means that the offset used to mitigate donation attacks will also be zero, nullifying its intended protection.

### 3. **Weak Protection in Some Cases**
   - A `decimalOffset` of `0` results in an added value of `1`, offering no real defense.
   - A `decimalOffset` of `1` or `2` introduces a minor buffer but still allows donation attacks.
   - The inconsistency makes the protocol vulnerable in some cases and overly strict in others.


## Impact

The protocol includes a check that reverts deposits if they result in zero shares, preventing donation attacks. However, this allows attackers to create a denial-of-service (DoS) scenario for users with small balances by increasing the minimum asset deposit required to receive at least one share. This issue arises because, for tokens with 18 decimals, no effective offset is applied, making donations inexpensive.

## Proof of concept


In the `createZlpVaults` function used for testing, the `decimalsOffset` is determined by `Constants.SYSTEM_DECIMALS - vaultsConfig[i].decimals`:  


https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/script/vaults/Vaults.sol#L109

```solidity
function createZlpVaults(address marketMakingEngine, address owner, uint256[2] memory vaultsIdsRange) public {
        // ****
        
            // deploy zlp vault as an upgradeable proxy
            address zlpVaultImpl = address(new ZlpVault());
            bytes memory zlpVaultInitData = abi.encodeWithSelector(
                ZlpVault.initialize.selector,
                marketMakingEngine,
                Constants.SYSTEM_DECIMALS - vaultsConfig[i].decimals,
                owner,
                IERC20(vaultAsset),
                vaultsConfig[i].vaultId
            );
        // ****
    }
```

For an asset with 18 decimals:  

- The `decimalsOffset` in the vault is calculated as `Constants.SYSTEM_DECIMALS - 18 = 0`.  
- The `indexToken` decimals become `18 + 0 = 18`.  
- The `decimalOffset` used for donation protection is `18 - 18 = 0`.  

This means that for assets with 18 decimals, no effective offset is applied, making donation attacks inexpensive and reducing the protocol’s protection against such exploits.


## Proposed Solution: Fixed Decimal Offset

To ensure **consistent** and **effective** mitigation of donation attacks, we propose replacing the dynamic `decimalOffset` with a fixed value.

### Implementation of Fixed Offset
Instead of computing `decimalOffset` dynamically, define it as a constant:

```solidity
uint8 constant FIXED_DECIMAL_OFFSET = 6; // Chosen based on attack resistance and usability trade-offs
```

**Note:** An offset of `3` forces an attacker to make a donation 1,000 times as large.

Then modify the calculation as follows:

```solidity
uint256 previewAssetsOut = sharesIn.mulDiv(
    totalAssetsMinusVaultDebt,
    IERC4626(vault.indexToken).totalSupply() + 10 ** FIXED_DECIMAL_OFFSET,
    MathOpenZeppelin.Rounding.Floor
);
```

---
### Example 3

**Auto Label:** Inaccurate price or amount scaling due to improper decimal handling leads to severe mispricing, over-withdrawals, and validation failures, resulting in fund loss or blocked withdrawals.  

**Original Text Preview:**

## Summary

The function `onAfterRemoveLiquidity` is responsible for handling post-liquidity removal operations, including fee calculations and administering liquidity back to the pool or QuantAMMAdmin. However, there is a subtle miscalculation in the `maxAmountsIn` parameter in line [541](https://github.com/Cyfrin/2024-12-quantamm/blob/a775db4273eb36e7b4536c5b60207c9f17541b92/pkg/pool-hooks/contracts/hooks-quantamm/UpliftOnlyExample.sol#L541) during admin fee processing, particularly when `localData.adminFeePercent` > 0

## Vulnerability Details

When the admin fee is calculated and accrued fees are intended to be added back to the liquidity pool, the value of `maxAmountsIn` becomes slightly less than the actual token amounts required. This leads to a mismatch during the `addLiquidity` operation in the `_vault`, which results in a reversion with the following error:

```solidity
AmountInAboveMax(token, amountInRaw, params.maxAmountsIn[i])
```

This issue prevents liquidity removal operations from completing successfully, effectively locking user funds in the pool.

The problem lies in the following block of code:

```solidity
        if (localData.adminFeePercent > 0) {
            _vault.addLiquidity(
                AddLiquidityParams({
                    pool: localData.pool,
                    to: IUpdateWeightRunner(_updateWeightRunner).getQuantAMMAdmin(),
                    maxAmountsIn: localData.accruedQuantAMMFees,
                    minBptAmountOut: localData.feeAmount.mulDown(localData.adminFeePercent) / 1e18,
                    kind: AddLiquidityKind.PROPORTIONAL,
                    userData: bytes("")
                })
            );
            emit ExitFeeCharged(
                userAddress,
                localData.pool,
                IERC20(localData.pool),
                localData.feeAmount.mulDown(localData.adminFeePercent) / 1e18
            );
        }
```

Here:

* `localData.accruedQuantAMMFees[i]` is calculated using:

```solidity
localData.accruedQuantAMMFees[i] = exitFee.mulDown(localData.adminFeePercent);
```

However, as vault calculation favours the vault over the user, the calculated `amountInRaw` in vault is slightly more than the `params.maxAmountsIn[i]`. Ref-[1](https://github.com/balancer/balancer-v3-monorepo/blob/93bacf3b5f219edff6214bcf58f8fe62ec3fde33/pkg/vault/contracts/Vault.sol#L712-L715)

* When the `_vault` validates the input via:

```solidity
if (amountInRaw > params.maxAmountsIn[i]) {
    revert AmountInAboveMax(token, amountInRaw, params.maxAmountsIn[i]);
}
```

The slight discrepancy causes a reversion, even when the difference is only by 1 wei.

## PoC

1. Create a new file named `UpLiftHookTest.t.sol` in `pkg/pool-hooks/test/foundry/`
2. Paste the following code:

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.24;

import { BaseVaultTest } from "@balancer-labs/v3-vault/test/foundry/utils/BaseVaultTest.sol";
import { QuantAMMWeightedPool, IQuantAMMWeightedPool } from "pool-quantamm/contracts/QuantAMMWeightedPool.sol";
import {QuantAMMWeightedPoolFactory} from "pool-quantamm/contracts/QuantAMMWeightedPoolFactory.sol";
import { UpdateWeightRunner, IUpdateRule } from "pool-quantamm/contracts/UpdateWeightRunner.sol";
import { MockChainlinkOracle } from "./utils/MockOracle.sol";
import "@balancer-labs/v3-interfaces/contracts/pool-quantamm/OracleWrapper.sol";
import { IUpdateRule } from "pool-quantamm/contracts/rules/UpdateRule.sol";
import { MockMomentumRule } from "pool-quantamm/contracts/mock/mockRules/MockMomentumRule.sol";
import { UpliftOnlyExample } from "../../contracts/hooks-quantamm/UpliftOnlyExample.sol";
import { IPermit2 } from "permit2/src/interfaces/IPermit2.sol";
import { IVault } from "@balancer-labs/v3-interfaces/contracts/vault/IVault.sol";
import { IVaultAdmin } from "@balancer-labs/v3-interfaces/contracts/vault/IVaultAdmin.sol";
import { PoolRoleAccounts, TokenConfig, HooksConfig } from "@balancer-labs/v3-interfaces/contracts/vault/VaultTypes.sol";
import { IWETH } from "@balancer-labs/v3-interfaces/contracts/solidity-utils/misc/IWETH.sol";
import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import { Router } from "@balancer-labs/v3-vault/contracts/Router.sol";
import {console} from "forge-std/console.sol";

contract UpLiftHookTest is BaseVaultTest {  // use default dai, usdc, weth and mock oracle
    //address daiOnETH = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
    //address usdcOnETH = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    //address wethOnETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    uint256 internal daiIdx;
    uint256 internal usdcIdx;

    uint256 SWAP_FEE_PERCENTAGE = 10e16;

    address quantAdmin = makeAddr("quantAdmin");
    address owner = makeAddr("owner");
    address poolCreator = makeAddr("poolCreator");
    address liquidityProvider1 = makeAddr("liquidityProvider1");
    address liquidityProvider2 = makeAddr("liquidityProvider2");
    address attacker = makeAddr("attacker");
    //address usdcUsd = 0x8fFfFfd4AfB6115b954Bd326cbe7B4BA576818f6;
    //address daiUsd = 0xAed0c38402a5d19df6E4c03F4E2DceD6e29c1ee9;
    //address ethOracle = 0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419;

    QuantAMMWeightedPool public weightedPool;
    QuantAMMWeightedPoolFactory public weightedPoolFactory;
    UpdateWeightRunner public updateWeightRunner;
    MockChainlinkOracle mockOracledai;
    MockChainlinkOracle mockOracleusdc;
    MockChainlinkOracle ethOracle;
    Router externalRouter;

    UpliftOnlyExample upLifthook;

    function setUp() public override {
        vm.warp(block.timestamp + 3600);
        mockOracledai = new MockChainlinkOracle(1e18, 0);
        mockOracleusdc = new MockChainlinkOracle(1e18, 0);
        ethOracle = new MockChainlinkOracle(2000e18, 0);
        updateWeightRunner = new UpdateWeightRunner(quantAdmin, address(ethOracle));

        vm.startPrank(quantAdmin);
        updateWeightRunner.addOracle(OracleWrapper(address(mockOracledai)));
        updateWeightRunner.addOracle(OracleWrapper(address(mockOracleusdc)));
        vm.stopPrank();

        super.setUp();

        (daiIdx, usdcIdx) = getSortedIndexes(address(dai), address(usdc));

        vm.prank(quantAdmin);
        updateWeightRunner.setApprovedActionsForPool(pool, 2);
    }

    function createHook() internal override returns (address) {
        // Create the factory here, because it needs to be deployed after the Vault, but before the hook contract.
        weightedPoolFactory = new QuantAMMWeightedPoolFactory(IVault(address(vault)), 365 days, "Factory v1", "Pool v1", address(updateWeightRunner));
        // lp will be the owner of the hook. Only LP is able to set hook fee percentages.
        vm.prank(quantAdmin);
        upLifthook = new UpliftOnlyExample(IVault(address(vault)), IWETH(weth), IPermit2(permit2), 100, 100, address(updateWeightRunner), "version 1", "lpnft", "LP-NFT");
        return address(upLifthook);
    }

    function _createPool(
        address[] memory tokens,
        string memory label
    ) internal override returns (address newPool, bytes memory poolArgs) {
        QuantAMMWeightedPoolFactory.NewPoolParams memory poolParams = _createPoolParams(tokens);

        (newPool, poolArgs) = weightedPoolFactory.create(poolParams);
        vm.label(newPool, label);

        authorizer.grantRole(vault.getActionId(IVaultAdmin.setStaticSwapFeePercentage.selector), quantAdmin);
        vm.prank(quantAdmin);
        vault.setStaticSwapFeePercentage(newPool, SWAP_FEE_PERCENTAGE);
    }

    function _createPoolParams(address[] memory tokens) internal returns (QuantAMMWeightedPoolFactory.NewPoolParams memory retParams) {
        PoolRoleAccounts memory roleAccounts;

        uint64[] memory lambdas = new uint64[]();
        lambdas[0] = 0.2e18;
        
        int256[][] memory parameters = new int256[][]();
        parameters[0] = new int256[]();
        parameters[0][0] = 0.2e18;

        address[][] memory oracles = new address[][]();
        oracles[0] = new address[]();
        oracles[0][0] = address(mockOracledai);
        oracles[1] = new address[]();
        oracles[1][0] = address(mockOracleusdc);

        uint256[] memory normalizedWeights = new uint256[]();
        normalizedWeights[0] = uint256(0.5e18);
        normalizedWeights[1] = uint256(0.5e18);

        IERC20[] memory ierctokens = new IERC20[]();
        for (uint256 i = 0; i < tokens.length; i++) {
            ierctokens[i] = IERC20(tokens[i]);
        }

        int256[] memory initialWeights = new int256[]();
        initialWeights[0] = 0.5e18;
        initialWeights[1] = 0.5e18;
        
        int256[] memory initialMovingAverages = new int256[]();
        initialMovingAverages[0] = 0.5e18;
        initialMovingAverages[1] = 0.5e18;
        
        int256[] memory initialIntermediateValues = new int256[]();
        initialIntermediateValues[0] = 0.5e18;
        initialIntermediateValues[1] = 0.5e18;
        
        TokenConfig[] memory tokenConfig = vault.buildTokenConfig(ierctokens);
        
        retParams = QuantAMMWeightedPoolFactory.NewPoolParams(
            "Pool With Donation",
            "PwD",
            tokenConfig,
            normalizedWeights,
            roleAccounts,
            0.02e18,
            address(poolHooksContract),
            true,
            true, // Do not disable unbalanced add/remove liquidity
            0x0000000000000000000000000000000000000000000000000000000000000000,
            initialWeights,
            IQuantAMMWeightedPool.PoolSettings(
                ierctokens,
                IUpdateRule(new MockMomentumRule(owner)),
                oracles,
                60,
                lambdas,
                0.2e18,
                0.2e18,
                0.3e18,
                parameters,
                poolCreator
            ),
            initialMovingAverages,
            initialIntermediateValues,
            3600,
            16,//able to set weights
            new string[][]()
        );
    }

    function testRemoveLiquidityUplift() public {
        addLiquidity();
        
        uint256[] memory minAmountsOut = new uint256[]();
        minAmountsOut[0] = 1;
        minAmountsOut[1] = 1;

        vm.prank(liquidityProvider1);
        UpliftOnlyExample(payable(poolHooksContract)).removeLiquidityProportional(2e18, minAmountsOut, true, pool);
    }

    function addLiquidity() public {
        deal(address(dai), liquidityProvider1, 100e18);
        deal(address(usdc), liquidityProvider1, 100e18);

        uint256[] memory maxAmountsIn = new uint256[]();
        maxAmountsIn[0] = 2.1e18;
        maxAmountsIn[1] = 2.1e18;
        uint256 exactBptAmountOut = 2e18;

        vm.startPrank(liquidityProvider1);
        IERC20(address(dai)).approve(address(permit2), 100e18);
        IERC20(address(usdc)).approve(address(permit2), 100e18);
        permit2.approve(address(dai), address(poolHooksContract), 100e18, uint48(block.timestamp));
        permit2.approve(address(usdc), address(poolHooksContract), 100e18, uint48(block.timestamp));
        UpliftOnlyExample(payable(poolHooksContract)).addLiquidityProportional(pool, maxAmountsIn, exactBptAmountOut, false, abi.encodePacked(liquidityProvider1));
        vm.stopPrank();

        deal(address(dai), liquidityProvider2, 100e18);
        deal(address(usdc), liquidityProvider2, 100e18);

        vm.startPrank(liquidityProvider2);
        IERC20(address(dai)).approve(address(permit2), 100e18);
        IERC20(address(usdc)).approve(address(permit2), 100e18);
        permit2.approve(address(dai), address(poolHooksContract), 100e18, uint48(block.timestamp));
        permit2.approve(address(usdc), address(poolHooksContract), 100e18, uint48(block.timestamp));
        UpliftOnlyExample(payable(poolHooksContract)).addLiquidityProportional(pool, maxAmountsIn, exactBptAmountOut, false, abi.encodePacked(liquidityProvider2));
        vm.stopPrank();

        console.log("Liquidity added");
    }
}
```

1. Go the the `pool-hooks` folder using terminal and run `forge test --mt testRemoveLiquidityUplift -vv`
   The test will revert with error:

```solidity
AmountInAboveMax()
```

## Impact

All liquidity removal operations revert, effectively locking user funds in the pool.

## Tools Used

Foundry

## Recommendations

* Add a small buffer to `localData.accruedQuantAMMFees` to account for precision errors. For example:

```solidity
for(uint256 i=0; i<localData.accruedQuantAMMFees.length; i++) {
    localData.accruedQuantAMMFees[i] += 1;
    hookAdjustedAmountsOutRaw[i] -= 1;
}
```

* Or reduce the `minBptAmountOut` silghtly:

```diff
        if (localData.adminFeePercent > 0) {
            _vault.addLiquidity(
                AddLiquidityParams({
                    pool: localData.pool,
                    to: IUpdateWeightRunner(_updateWeightRunner).getQuantAMMAdmin(),
                    maxAmountsIn: localData.accruedQuantAMMFees,
-                   minBptAmountOut: localData.feeAmount.mulDown(localData.adminFeePercent) / 1e18,
+                   minBptAmountOut: localData.feeAmount.mulDown(localData.adminFeePercent) / 1e18 -1,
                    kind: AddLiquidityKind.PROPORTIONAL,
                    userData: bytes("")
                })
            );
            emit ExitFeeCharged(
                userAddress,
                localData.pool,
                IERC20(localData.pool),
                localData.feeAmount.mulDown(localData.adminFeePercent) / 1e18
            );
        }
```

---
