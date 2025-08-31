# Cluster -1149

**Rank:** #324  
**Count:** 21  

## Label
Failure to properly validate or update Total Collateral Ratio (TCR) during redemptions enables cascading TCR drops, silent mode transitions, and invalid invariants, leading to systemic instability and griefing attacks.

## Cluster Information
- **Total Findings:** 21

## Examples

### Example 1

**Auto Label:** Failure to properly validate or update Total Collateral Ratio (TCR) during redemptions enables cascading TCR drops, silent mode transitions, and invalid invariants, leading to systemic instability and griefing attacks.  

**Original Text Preview:**

**Impact**

`RedemptionOperations` checks the `TCR < MCR`, but should most likely check for `TCR < CCR`

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/RedemptionOperations.sol#L101-L103

```solidity
    (, uint TCR, , ) = storagePool.checkRecoveryMode(vars.priceCache); /// @audit High? not checking RM -> Mint for free, Inflate total supply (pay no redemption), redeem at small fee
    if (TCR < MCR) revert LessThanMCR(); /// @audit-ok force to liquidate if all system is insolvent

```

This is because during Recovery Mode, minting fees are voided, meaning that the system may open up to additional arbitrages via Redemptions

Redemptions are generally disabled during RM in favour of liquidations


**Mitigation**

Investigate if additional arbitrages could be detrimental to your protocol due to reduced minting fees

From checking liquity they use a check similar to yours:
https://github.com/liquity/dev/blob/e38edf3dd67e5ca7e38b83bcf32d515f896a7d2f/packages/contracts/contracts/TroveManager.sol#L948-L962

---
### Example 2

**Auto Label:** Failure to properly validate or update Total Collateral Ratio (TCR) during redemptions enables cascading TCR drops, silent mode transitions, and invalid invariants, leading to systemic instability and griefing attacks.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

When the redemption cooldown requirement is set, users need to escrow their stables for a certain period before being able to redeem collateral. After the cooldown period, the grace period starts, and users can redeem their collateral. However, if the redemption does not take place before the grace period ends, the user will be charged a penalty fee.

Given that the `redeemCollateral` function reverts if the TCR is below the MCR, if the TCR falls below the MCR during the cooldown period and does not recover before the grace period ends, users will be unable to redeem their collateral and will be charged a penalty.

```solidity
File: PositionManager.sol

433:        _requireTCRoverMCR(totals.price, MCR);
```

## Recommendations

Allow users to dequeue their escrowed stables when TCR < MCR.

---
### Example 3

**Auto Label:** Improper scaling and missing balance updates lead to incorrect collateral transfers and inflated rewards, causing protocol fund loss and failed liquidations.  

**Original Text Preview:**

First of all, in `CDPVault`, the amount of `collateral` maintained in each position is scaled using `tokenScale`. See the code in `deposit`:

https://github.com/code-423n4/2024-07-loopfi/blob/57871f64bdea450c1f04c9a53dc1a78223719164/src/CDPVault.sol#L223-L233

and the function `modifyCollateralAndDebt()`:

https://github.com/code-423n4/2024-07-loopfi/blob/57871f64bdea450c1f04c9a53dc1a78223719164/src/CDPVault.sol#L367-L460

For example, when withdrawing collateral, it will scale it with `tokenScale` from internal amount:

```javascript
  uint256 amount = wmul(abs(deltaCollateral), tokenScale);
            token.safeTransfer(collateralizer, amount);
```

However, when sending collateral to the liquidator, it uses the internal amount without scaling by `tokenScale` in function `liquidatePosition` at L565.

```javascript
 token.safeTransfer(msg.sender, takeCollateral);
```

As a result, when `tokenScale < 10 ** 18`, the above line actually send more tokens to the liquidator than it is supposed to, a loss of funds for the protocol.

In the following POC, we show:

1. We use a collateral token that has 16 decimals, as a a result, `tokenScale = 19 ** 16`.
2. Frank deposits 1M units of collateral.
3. The test contract deposits 100 units of collateral and then borrows 10 ether of underlying tokens, with the price of collateral being 1 ether.
4. The price of the collateral drops to 0.1 ether.
5. Kathy liquidates the position of the test contract with 1 ether of underlying tokens.
6. Kathy is supposed to receive 10.52 units of collateral; however, she receives 1052 units of collateral instead due to the above bug. This amount is much greater than the collateral for the position held by the test contract.
7. The protocol loses collateral to the liquidator, in particular, the collateral that is supposed to be owned by Frank.
8. Now, when Frank tries to withdraw his collateral, he fails.

Please run `forge test --match-test testLiquidate1 -vv`:

<details>

```javascript
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.19;

import "forge-std/console2.sol";
import {TestBase, ERC20PresetMinterPauser} from "../TestBase.sol";

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {IERC20Metadata} from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";

import {IOracle} from "../../interfaces/IOracle.sol";
import {ICDPVaultBase} from "../../interfaces/ICDPVault.sol";
import {CDPVaultConstants, CDPVaultConfig} from "../../interfaces/ICDPVault.sol";
import {IPermission} from "../../interfaces/IPermission.sol";

import {WAD, wmul, wdiv, wpow, toInt256} from "../../utils/Math.sol";
import {CDPVault, VAULT_CONFIG_ROLE} from "../../CDPVault.sol";
import {console} from "forge-std/console.sol";
import {StdCheats} from "forge-std/StdCheats.sol";

contract MockTokenScaled is ERC20PresetMinterPauser {
    uint8 private _decimals;

    constructor(string memory name, string memory symbol, uint8 decimals_) ERC20PresetMinterPauser(name, symbol) {
        _decimals = decimals_;
    }

    function decimals() public view override returns (uint8) {
        return _decimals;
    }
}
import {CDPVault, VAULT_CONFIG_ROLE} from "../../CDPVault.sol";
import {console} from "forge-std/console.sol";

contract CDPVaultWrapper is CDPVault {
    constructor(CDPVaultConstants memory constants, CDPVaultConfig memory config) CDPVault(constants, config) {}
}

contract PositionOwner {
    constructor(IPermission vault) {
        // Allow deployer to modify Position
        vault.modifyPermission(msg.sender, true);
    }
}

contract CDPVaultTest is TestBase {
    MockTokenScaled tokenScaled;

    /*//////////////////////////////////////////////////////////////
                            HELPER FUNCTIONS
    //////////////////////////////////////////////////////////////*/

    function _depositCollateral(CDPVault vault, uint256 amount) internal {
        token.mint(address(this), amount);
        (uint256 collateralBefore, , , , , ) = vault.positions(address(this));
        token.approve(address(vault), amount);
        vault.deposit(address(this), amount);
        (uint256 collateralAfter, , , , , ) = vault.positions(address(this));
        assertEq(collateralAfter, collateralBefore + amount);
    }

    function _modifyCollateralAndDebt(CDPVault vault, int256 collateral, int256 debt) internal {
        if (debt < 0) {
            mockWETH.mint(address(this), uint256(-debt));
            mockWETH.approve(address(vault), uint256(-debt));
        }

        if (collateral > 0) {
            token.mint(address(this), uint256(collateral));
            token.approve(address(vault), uint256(collateral));
        }

        (uint256 collateralBefore, uint256 debtBefore, , , , ) = vault.positions(address(this));
        uint256 virtualDebtBefore = virtualDebt(vault, address(this));
        uint256 vaultCreditBefore = credit(address(this));

        vault.modifyCollateralAndDebt(address(this), address(this), address(this), collateral, debt);
        {
            (uint256 collateralAfter, uint256 debtAfter, , , , ) = vault.positions(address(this));
            assertEq(toInt256(collateralAfter), toInt256(collateralBefore) + collateral);
            assertEq(toInt256(debtAfter), toInt256(debtBefore) + debt);
        }

        uint256 virtualDebtAfter = virtualDebt(vault, address(this));
        int256 deltaDebt = toInt256(virtualDebtAfter) - toInt256(virtualDebtBefore);
        {
            uint256 tokensAfter = credit(address(this));
            assertEq(toInt256(tokensAfter), toInt256(vaultCreditBefore) + deltaDebt);
        }

        uint256 vaultCreditAfter = credit(address(this));
        assertEq(toInt256(vaultCreditBefore + virtualDebtAfter), toInt256(vaultCreditAfter + virtualDebtBefore));
        assertEq(toInt256(vaultCreditBefore + virtualDebtAfter), toInt256(vaultCreditAfter + virtualDebtBefore));
    }

    function _updateSpot(uint256 price) internal {
        oracle.updateSpot(address(token), price);
    }

    function _collateralizationRatio(CDPVault vault) internal view returns (uint256) {
        (uint256 collateral, , , , , ) = vault.positions(address(this));
        if (collateral == 0) return type(uint256).max;
        return wdiv(wmul(collateral, vault.spotPrice()), virtualDebt(vault, address(this)));
    }

    function _createVaultWrapper(uint256 liquidationRatio) private returns (CDPVaultWrapper vault) {
        CDPVaultConstants memory constants = _getDefaultVaultConstants();
        CDPVaultConfig memory config = _getDefaultVaultConfig();
        config.liquidationRatio = uint64(liquidationRatio);

        vault = new CDPVaultWrapper(constants, config);
    }

    function _setDebtCeiling(CDPVault vault, uint256 debtCeiling) internal {
        // cdm.setParameter(address(vault), "debtCeiling", debtCeiling);
        liquidityPool.setCreditManagerDebtLimit(address(vault), debtCeiling);
    }

    
    function printPosition(CDPVault vault, address p, string memory name) public{
        console2.log("\n =================================================");
        console2.log("position infor for ", name);

        (uint256 collateral, // [wad]
        uint256 debt, // [wad]
        uint256 lastDebtUpdate, // [timestamp]
        uint256 cumulativeIndexLastUpdate,
        uint192 cumulativeQuotaIndexLU,
        uint128 cumulativeQuotaInterest
        ) = vault.positions(p);

        console2.log("collateral: ", collateral);
        console2.log("debt: ", debt);
        console2.log("cumulativeQuotaInterest: ", cumulativeQuotaInterest);
        console2.log("lastUpdate: ", lastDebtUpdate);
        
        uint256 cumulativeIndexNow = liquidityPool.baseInterestIndex();
        uint256 cumulativeQuotaIndexNow = quotaKeeper.cumulativeIndex(address(tokenScaled));
        console2.log("cumulatveIndexNow: ", cumulativeIndexNow);
        console2.log("cumulativeIndexLastUpdate:", cumulativeIndexLastUpdate);
        console2.log("cumulativeQuotaIndexNow: ", cumulativeQuotaIndexNow);
        console2.log("cumulativeQuotaIndexLU: ", cumulativeQuotaIndexLU);
        console2.log("=================================================\n ");
    }

    
    function printBalances(address a, string memory name) public{
        console2.log("\n =================================================");
        console2.log("Balances for ", name);
        console2.log("Collateral balance: ", token.balanceOf(a));
        console2.log("borrow token balance: ", mockWETH.balanceOf(a));
        console2.log("=================================================\n ");
    }


function testLiquidate1() public{
        address Frank = makeAddr("Frank");
        address Kathy = makeAddr("Kathy");
        
        

        tokenScaled = new MockTokenScaled("TestToken", "TST", 16);  // 16 decimals

        CDPVault vault = createCDPVault(tokenScaled, 150 ether, 0, 1.25 ether, 1.0 ether, 0.95 ether);
        createGaugeAndSetGauge(address(vault), address(tokenScaled));

        // frank does a deposit 1M units
        tokenScaled.mint(Frank, 1000000*10**16);
        vm.startPrank(Frank);
        tokenScaled.approve(address(vault), 1000000*10**16); // 100 units
        vault.deposit(Frank, 1000000*10**16);
        vm.stopPrank();

        // this test contract does a deposit 100 units
        tokenScaled.mint(address(this), 100*10**16);
        tokenScaled.approve(address(vault), 100*10**16); // 100 units
        vault.deposit(address(this), 100*10**16);

        oracle.updateSpot(address(tokenScaled), 1 ether);
        vault.borrow(address(this), address(this), 10 ether);    // 10 ether debt, 100 units of collateral
         
        
        console2.log("\n \n --------------------liquidate now -----------------");
        oracle.updateSpot(address(tokenScaled), 0.1 ether);
        
        mockWETH.mint(Kathy, 1 ether);
        vm.startPrank(Kathy);
        mockWETH.approve(address(vault), 1 ether);
        vault.liquidatePosition(address(this), 1 ether);
        vm.stopPrank();

        printPosition(vault, address(this), "this position");
        console2.log("done");

        vm.startPrank(Frank);
        vm.expectRevert();  // not enough collatreral to withdraw now
        vault.withdraw(Frank, 1000000*10**16);
        vm.stopPrank();
}
}
```

</details>

### Tools Used

Foundry

### Recommended Mitigation Steps

Scale the collateral amount from internal representation to the real amount by `tokenScale`.

### Assessed type

Decimal

**[0xtj24 (LoopFi) acknowledged](https://github.com/code-423n4/2024-07-loopfi-findings/issues/223#event-14195872290)**

**[Koolex (judge) decreased severity to Medium](https://github.com/code-423n4/2024-07-loopfi-findings/issues/223#issuecomment-2442402623)**

*Note: For full discussion, see [here](https://github.com/code-423n4/2024-07-loopfi-findings/issues/223).*

***

---
