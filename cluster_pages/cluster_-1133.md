# Cluster -1133

**Rank:** #261  
**Count:** 33  

## Label
Mismanagement of liquidity and asset pricing due to flawed assumptions, inadequate fee mechanisms, and lack of real-time balance validation enables malicious actors to manipulate token values, extract value, or cause denial-of-service through arbitrage and front-running.

## Cluster Information
- **Total Findings:** 33

## Examples

### Example 1

**Auto Label:** Insufficient input validation and missing fee enforcement lead to unauthorized parameter manipulation, underpayment, or spam, enabling denial-of-service, incorrect state changes, or economic exploitation.  

**Original Text Preview:**

[/contracts/pool-manager/src/helpers.rs# L563-L564](https://github.com/code-423n4/2024-11-mantra-dex/blob/26714ea59dab7ecfafca9db1138d60adcf513588/contracts/pool-manager/src/helpers.rs# L563-L564)

### Finding description and impact

`create_pool` is permissionless. User will need to pay both pool creation fee and token factory fee (fees charged for creating lp token) when creating a pool.

The vulnerabilities are:

1. In `validate_fees_are_paid`, a strict equality check is used between total paid fees in `pool_creation` token and required pool creation fee. `paid_pool_fee_amount == pool_creation_fee.amount`.
2. `denom_creation_fee.iter()` uses `.any` instead of `.all`, which allows user to only pay one coin from `denom_creation`.
```

//contracts/pool-manager/src/helpers.rs
pub fn validate_fees_are_paid(
    pool_creation_fee: &Coin,
    denom_creation_fee: Vec<Coin>,
    info: &MessageInfo,
) -> Result<Vec<Coin>, ContractError> {
...
    // Check if the pool fee denom is found in the vector of the token factory possible fee denoms
    if let Some(tf_fee) = denom_creation_fee
        .iter()
        .find(|fee| &fee.denom == pool_fee_denom)
    {
        // If the token factory fee has only one option, check if the user paid the sum of the fees
        if denom_creation_fee.len() == 1usize {
...
        } else {
            // If the token factory fee has multiple options besides pool_fee_denom, check if the user paid the pool creation fee
            let paid_pool_fee_amount = get_paid_pool_fee_amount(info, pool_fee_denom)?;
            //@audit (1) strict equality check. When user is also required to pay denom_creation_fee in pool creation fee token, check will revert create_pool
            ensure!(
 |>             paid_pool_fee_amount == pool_creation_fee.amount,
                ContractError::InvalidPoolCreationFee {
                    amount: paid_pool_fee_amount,
                    expected: pool_creation_fee.amount,
                }
            );
...
            // Check if the user paid the token factory fee in any other of the allowed denoms
            //@audit (2) iter().any() only requires one of denom_creation_fee token to be paid.
|>          let tf_fee_paid = denom_creation_fee.iter().any(|fee| {
                let paid_fee_amount = info
                    .funds
                    .iter()
                    .filter(|fund| fund.denom == fee.denom)
                    .map(|fund| fund.amount)
                    .try_fold(Uint128::zero(), |acc, amount| acc.checked_add(amount))
                    .unwrap_or(Uint128::zero());

                total_fees.push(Coin {
                    denom: fee.denom.clone(),
                    amount: paid_fee_amount,
                });

                paid_fee_amount == fee.amount
            });
...
```

[/contracts/pool-manager/src/helpers.rs# L577](https://github.com/code-423n4/2024-11-mantra-dex/blob/26714ea59dab7ecfafca9db1138d60adcf513588/contracts/pool-manager/src/helpers.rs# L577)

Based on cosmwasm tokenfactory, denom creation fee (`std.coins`) can contain multiple coins and every coin needs to be paid.
```

//x/tokenfactory/simulation/operations.go
func SimulateMsgCreateDenom(tfKeeper TokenfactoryKeeper, ak types.AccountKeeper, bk BankKeeper) simtypes.Operation {
...
		// Check if sims account enough create fee
		createFee := tfKeeper.GetParams(ctx).DenomCreationFee
		balances := bk.GetAllBalances(ctx, simAccount.Address)
|>		_, hasNeg := balances.SafeSub(createFee) //@audit-info all denom creation fee tokens have to be paid
		if hasNeg {
			return simtypes.NoOpMsg(types.ModuleName, types.MsgCreateDenom{}.Type(), "Creator not enough creation fee"), nil, nil
		}
...
```

<https://github.com/CosmWasm/token-factory/blob/47dc2d5ae36980bcc03cf746580f7cb3deabc39e/x/tokenfactory/simulation/operations.go# L359-L361>

Flows: `contracts/pool-manager/src/manager/commands::create_pool -> validate_fees_are_paid()`

### Impact

User can either underpay fees, or `create_pool` tx will revert.

### Proof of Concept

Because pool creation fee and all denom creation fee tokens need to be paid. Consider this edge case:
A. There are more than one token factory fee tokens required in `denom_creation_fee`.
B. One of the denom*creation fee token is the pool creation fee token (`fee.denom == pool*creation\_fee.denom`).

In this case, user is required to send `=` pool token amount (`pool_creation_fee + denom_creation_fee`) and all other denom creation token amount.

If user sends all the required fees, `paid_pool_fee_amount == pool_creation_fee.amount` check will fail because `paid_pool_fee.amount > pool_creation_fee.amount`. This reverts `create_pool`.

If user chooses to take advantage of `denom_creation_fee.iter().any`, user sent `=` `pool_creation_fee` `+` one of denom creation token amounts (not the pool token). This passes both the strict equality check and `.any`. However, the user only paid `pool_creation_fee` and underpaid `denom_creation_fee`.

### Recommended mitigation steps

1. Change `.any()` -> `.all()`.
2. Because this branch `denom_creation_fee` contains the `pool_creation` token, needs to add a control flow to handle the iteration of `pool_creation` token to check `paid_fee_amount == fee.amount + pool_creation_fee.amount`.

**[3docSec (judge) commented](https://code4rena.com/audits/2024-11-mantra-dex/submissions/F-4?commentParent=xoKfp8pBSFr):**

> Looks to be intended behavior, as per comment L576.

**[jvr0x (MANTRA) disputed and commented](https://code4rena.com/audits/2024-11-mantra-dex/submissions/F-4?commentParent=qek9ytnY4yb):**

> `pool_creation_fee` is the fee for creating the pool while the vec.
> `denom_creation_fee` are the tokens that the user can pay for the token factory in. Only 1 is enough, no need to pay in all the denoms listed there.

**[3docSec (judge) commented](https://code4rena.com/audits/2024-11-mantra-dex/submissions/F-4?commentParent=MWSVxoDFQzc):**

> Behavior is inconsistent with the MantraChain tokenfactory that collects all fees; okay for valid Medium.

---

---
### Example 2

**Auto Label:** Mismanagement of liquidity and asset pricing due to flawed assumptions, inadequate fee mechanisms, and lack of real-time balance validation enables malicious actors to manipulate token values, extract value, or cause denial-of-service through arbitrage and front-running.  

**Original Text Preview:**

The `removeStrategy` function in the `MultiStrategy` contract allows the removal of a strategy and redistributes the withdrawn funds among the remaining strategies.

* [Refered code](https://github.com/code-423n4/2024-12-bakerfi/blob/0daf8a0547b6245faed5b6cd3f5daf44d2ea7c9a/contracts/core/MultiStrategy.sol# L263C8-L266C10):

  
```

      if (strategyAssets > 0) {
          IStrategy(_strategies[index]).undeploy(strategyAssets);
          _allocateAssets(strategyAssets);
      }
  
```

The issue arises when the last strategy is removed. The weight (`_weights[index]`) of the last strategy is first subtracted from `_totalWeight`, which results in `_totalWeight` being zero, and it is then set to zero.

* [Vulnerable logic](https://github.com/code-423n4/2024-12-bakerfi/blob/0daf8a0547b6245faed5b6cd3f5daf44d2ea7c9a/contracts/core/MultiStrategy.sol# L259-L260):

  
```

      _totalWeight -= _weights[index];
      _weights[index] = 0;
  
```

Later, when `_allocateAssets` is called: for each of the active `_strategies` (the last strategy has not yet been removed), it attempts to calculate the fraction of the input amount. However, since `_totalWeight` is zero, the execution is reverted with a “panic: division or modulo by zero” error.

* [Vulnerable logic](https://github.com/code-423n4/2024-12-bakerfi/blob/0daf8a0547b6245faed5b6cd3f5daf44d2ea7c9a/contracts/core/MultiStrategy.sol# L143-L154):

  
```

  function _allocateAssets(uint256 amount) internal returns (uint256 totalDeployed) {
      totalDeployed = 0;
      for (uint256 i = 0; i < _strategies.length; ) {
          uint256 fractAmount = (amount * _weights[i]) / _totalWeight;
          if (fractAmount > 0) {
              totalDeployed += IStrategy(_strategies[i]).deploy(fractAmount);
          }
          unchecked {
              i++;
          }
      }
  }
  
```

### Impact

The `VAULT_MANAGER_ROLE` would be unable to delete the last strategy.

### Proof of Concept

This PoC follows these steps:

1. Deploy the `StrategyUniV3SwapAnd` strategy (or any other strategy) and initialize the `MultiStrategyVault` with this strategy to emulate the state where only one strategy remains in the vault.
2. Deposit a certain amount into the vault.
3. The `VAULT_MANAGER_ROLE`, in this case the same person as the `DEPLOYER`, attempts to remove the strategy.

However, the execution fails with:
```

    │   ├─ [109194] MultiStrategyVault::removeStrategy(0) [delegatecall]
...
    │   │   └─ ← [Revert] panic: division or modulo by zero (0x12)
    │   └─ ← [Revert] panic: division or modulo by zero (0x12)
```


```

// SPDX-License-Identifier: Unlicense
pragma solidity ^0.8.24;

import {Test} from "lib/forge-std/src/Test.sol";
import {console} from "lib/forge-std/src/console.sol";
import {MockERC20} from "lib/forge-std/src/mocks/MockERC20.sol";

import {Command} from "contracts/core/MultiCommand.sol";
import {MultiStrategy} from "contracts/core/MultiStrategy.sol";
import {MultiStrategyVault} from "contracts/core/MultiStrategyVault.sol";
import {BakerFiProxy} from "contracts/proxy/BakerFiProxy.sol";
import {BakerFiProxyAdmin} from "contracts/proxy/BakerFiProxyAdmin.sol";
import {StrategyUniV3SwapAnd} from "contracts/core/strategies/StrategyUniV3SwapAnd.sol";
import {OracleMock} from "contracts/mocks/OracleMock.sol";
import {UniV3RouterMock} from "contracts/mocks/UniV3RouterMock.sol";
import {StrategyPark} from "contracts/core/strategies/StrategyPark.sol";
import {IStrategy} from "contracts/interfaces/core/IStrategy.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {UseUnifiedSwapper} from "contracts/core/hooks/swappers/UseUnifiedSwapper.sol";

contract PoCs is Test {
    address immutable DEPLOYER = makeAddr("DEPLOYER");
    address immutable USER1 = makeAddr("USER1");

    address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;

    address constant WstETH = 0x7f39C581F595B53c5cb19bD0b3f8dA6c935E2Ca0;

    function setUp() public {
        vm.label(WETH, "WETH");
        vm.label(WstETH, "WstETH");

        vm.createSelectFork("https://rpc.ankr.com/eth");
    }

    function test_removeStrategy_last_not_removed() public {
        vm.startPrank(DEPLOYER);
        address bakerFiProxyAdmin = address(new BakerFiProxyAdmin(DEPLOYER));

        // @note strategy deployment logic from test/core/strategies/StrategyUniV3SwapPark.ts
        uint256 WETH_SUPPLY = 100000 ether;
        deal(address(WETH), DEPLOYER, WETH_SUPPLY);

        uint256 WstETH_SUPPLY = 30000000 ether;
        deal(address(WstETH), DEPLOYER, WstETH_SUPPLY);

        OracleMock mockOracle = new OracleMock();
        mockOracle.setDecimals(18);
        mockOracle.setLatestPrice(1e16);

        UniV3RouterMock mockUniV3 = new UniV3RouterMock(
            IERC20(address(WETH)),
            IERC20(address(WstETH))
        );

        MockERC20(WETH).transfer(address(mockUniV3), WETH_SUPPLY);
        MockERC20(WstETH).transfer(address(mockUniV3), WstETH_SUPPLY);
        mockUniV3.setPrice(1e11);

        StrategyPark strategyPark = new StrategyPark(DEPLOYER, address(WstETH));

        StrategyUniV3SwapAnd strategyUniV3SwapAnd = new StrategyUniV3SwapAnd(
            DEPLOYER,
            IERC20(address(WETH)),
            strategyPark,
            mockOracle,
            mockUniV3,
            3000
        );

        strategyPark.transferOwnership(address(strategyUniV3SwapAnd));
        vm.label(address(strategyUniV3SwapAnd), "StrategyUniV3SwapAnd");
        // @note End of strategy deployment

        strategyUniV3SwapAnd.setMaxSlippage(1e8 /* 10% */);

        address impl = address(new MultiStrategyVault());

        IStrategy[] memory istrategies = new IStrategy[](1);
        istrategies[0] = strategyUniV3SwapAnd;
        uint16[] memory iweights = new uint16[](1);
        iweights[0] = 100;

        MultiStrategyVault msVault = MultiStrategyVault(
            payable(
                address(
                    new BakerFiProxy(
                        impl,
                        bakerFiProxyAdmin,
                        abi.encodeWithSelector(
                            MultiStrategyVault.initialize.selector,
                            DEPLOYER,
                            "AuditPoC",
                            "APC",
                            WETH,
                            istrategies,
                            iweights,
                            WETH
                        )
                    )
                )
            )
        );

        vm.startPrank(DEPLOYER);
        deal(address(WETH), DEPLOYER, 10 ether);
        MockERC20(WETH).approve(address(msVault), 10 ether);
        msVault.deposit(10 ether, DEPLOYER);

        msVault.removeStrategy(0);
    }
}
```

### Recommended Mitigation Steps

There are several ways to improve the code to fix the issue (such as adding a check for zero, etc.). However, the most straightforward and direct approach is to remove the strategy from `_strategies` before calling `_allocateAssets`:
```

    function removeStrategy(
        uint256 index
    ) external onlyRole(VAULT_MANAGER_ROLE) {
        // Validate the index to ensure it is within bounds
        if (index >= _strategies.length) revert InvalidStrategyIndex(index);

        // Retrieve the total assets managed by the strategy to be removed
        uint256 strategyAssets = _strategies[index].totalAssets();

        // Update the total weight and mark the weight of the removed strategy as zero
        _totalWeight -= _weights[index];
        _weights[index] = 0;

        IStrategy cache_strategy = _strategies[index];

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

        // If the strategy has assets, undeploy them and allocate accordingly
        if (strategyAssets > 0) {
            IStrategy(cache_strategy).undeploy(strategyAssets);
            _allocateAssets(strategyAssets);
        }
    }3
```

**chefkenji (BakerFi) confirmed**

**[0xpiken (warden) commented](https://code4rena.com/audits/2024-12-bakerfi-invitational/submissions/F-36?commentParent=iYzcHf4jfS7):**

> The last strategy should not be allowed to be removed since `MultiStrategyVault` allocates assets accordingly to its strategies.
> Removing the last strategy will leads DoS on `MultiStrategyVault`. The worse is that no user can withdraw their assets since `_deallocateAssets()` will return `totalUndeployed` as `0`.

**[pfapostol (warden) commented](https://code4rena.com/audits/2024-12-bakerfi-invitational/submissions/F-36?commentParent=iYzcHf4jfS7&commentChild=WshnvTh35Yi):**

> Yes, but that sounds more like a second problem, unrelated to this one.
> The order of operations in the function is clearly incorrect.

**[Dravee (judge) commented](https://code4rena.com/audits/2024-12-bakerfi-invitational/submissions/F-36?commentParent=iYzcHf4jfS7&commentChild=WSEU9MfhqXJ):**

> Per the sponsor:
>
> > I understand both points but for me is an issue because it leaves the vault on a weird state (funds are waiting for a rebalance) that could only be unlocked by the vault manager with a rebalance
>
> This finding is still valid.

**[BakerFi mitigated](https://github.com/code-423n4/2025-01-bakerfi-mitigation?tab=readme-ov-file# findings-being-mitigated):**

> [PR-18](https://github.com/baker-fi/bakerfi-contracts/pull/18)

**Status:** Mitigation confirmed. Full details in reports from [shaflow2](https://code4rena.com/evaluate/2025-01-bakerfi-mitigation-review/findings/S-14) and [0xlemon](https://code4rena.com/evaluate/2025-01-bakerfi-mitigation-review/findings/S-37).

---

---
### Example 3

**Auto Label:** Mismanagement of liquidity and asset pricing due to flawed assumptions, inadequate fee mechanisms, and lack of real-time balance validation enables malicious actors to manipulate token values, extract value, or cause denial-of-service through arbitrage and front-running.  

**Original Text Preview:**

<https://github.com/code-423n4/2024-12-bakerfi/blob/main/contracts/core/MultiStrategy.sol# L173>

### Finding description and impact

The `MultiStrategy# _deallocateAssets()` function will be DoSed if `IStrategy# undeploy(0)` is called.

### Proof of Concept

When withdrawing user assets from a multi-strategies vault, it will be withdrawn pro-rata from the strategies base on their deployed assets:
<https://github.com/code-423n4/2024-12-bakerfi/blob/main/contracts/core/MultiStrategyVault.sol>:
```

    function _undeploy(uint256 assets) internal virtual override returns (uint256 undeployedAmount) {
        undeployedAmount = _deallocateAssets(assets); // Deallocates assets from the strategies and returns the undeployed amount
    }
```

<https://github.com/code-423n4/2024-12-bakerfi/blob/main/contracts/core/MultiStrategy.sol>:
```

    function _deallocateAssets(uint256 amount) internal returns (uint256 totalUndeployed) {
        uint256[] memory currentAssets = new uint256[](_strategies.length);

        uint256 totalAssets = 0;
        uint256 strategiesLength = _strategies.length;

        for (uint256 i = 0; i < strategiesLength; i++) {
            currentAssets[i] = IStrategy(_strategies[i]).totalAssets();
            totalAssets += currentAssets[i];
        }
        totalUndeployed = 0;
        for (uint256 i = 0; i < strategiesLength; i++) {
            uint256 fractAmount = (amount * currentAssets[i]) / totalAssets;
            totalUndeployed += IStrategy(_strategies[i]).undeploy(fractAmount);
        }
    }
```

If a strategy has not yet deployed any assets, its `totalAssets()` call will return zero. Subsequently, the system will attempt to execute `IStrategy.undeploy(0)`. However, this call is likely to revert, potentially leading to a denial-of-service condition for the entire withdrawal function:
[StrategySupplyBase.sol](https://github.com/code-423n4/2024-12-bakerfi/blob/main/contracts/core/strategies/StrategySupplyBase.sol) reverts `ZeroAmount()` when `amount` is `0`:
```

    function undeploy(uint256 amount) external nonReentrant onlyOwner returns (uint256 undeployedAmount) {
@>      if (amount == 0) revert ZeroAmount();

        // Get Balance
        uint256 balance = getBalance();
        if (amount > balance) revert InsufficientBalance();

        // Transfer assets back to caller
        uint256 withdrawalValue = _undeploy(amount);

        // Check withdrawal value matches the initial amount
        // Transfer assets to user
        ERC20(_asset).safeTransfer(msg.sender, withdrawalValue);

        balance -= amount;
        emit StrategyUndeploy(msg.sender, withdrawalValue);
        emit StrategyAmountUpdate(balance);

        return amount;
    }
```

[StrategyLeverage.sol](https://github.com/code-423n4/2024-12-bakerfi/blob/main/contracts/core/strategies/StrategyLeverage.sol) reverts `NoCollateralMarginToScale()` because `totalCollateralInDebt` is equal to `totalDebtBalance`(both of them are `0`):
```

    function undeploy(uint256 amount) external onlyOwner nonReentrant returns (uint256 undeployedAmount) {
        undeployedAmount = _undeploy(amount, payable(msg.sender));
    }

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
@>      if (totalCollateralInDebt <= totalDebtBalance) revert NoCollateralMarginToScale();

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

Copy below codes to [VaultMultiStrategy.ts](https://github.com/code-423n4/2024-12-bakerfi/blob/main/test/core/vault/VaultMultiStrategy.ts) and run `npm run test`:
```

  it.only('Withdraw is DoSed', async () => {
    const { vault, usdc, owner, park1, park2 } = await loadFixture(deployMultiStrategyVaultFixture);
    const amount = 10000n * 10n ** 18n;
    await usdc.approve(vault.target, amount);
    await vault.deposit(amount, owner.address);

    //@audit-info deploy a erc4626 vault for StrategySupplyERC4626
    const ERC4626Vault = await ethers.getContractFactory("ERC4626VaultMock");
    const erc4626Vault = await ERC4626Vault.deploy(usdc.getAddress());
    await erc4626Vault.waitForDeployment();

    // Deploy StrategySupply contract
    const StrategySupply = await ethers.getContractFactory('StrategySupplyERC4626');
    const strategy = await StrategySupply.deploy(
      owner.address,
      await usdc.getAddress(),
      await erc4626Vault.getAddress(),
    );
    await strategy.waitForDeployment();
    await strategy.transferOwnership(await vault.getAddress());

    //@audit-info add the new strategy to vault
    await vault.addStrategy(await strategy.getAddress());
    //@audit-info the new strategy has been added
    expect(await vault.strategies()).to.include(await strategy.getAddress());
    //@audit-info withdrawal is DoSed
    await expect(vault.withdraw(amount, owner.address, owner.address)).to.be.revertedWithCustomError(
      strategy,
      'ZeroAmount',
    );
  });
```

### Recommended mitigation steps

Check if the amount is `0` before undeploying it:
```

    function _deallocateAssets(uint256 amount) internal returns (uint256 totalUndeployed) {
        uint256[] memory currentAssets = new uint256[](_strategies.length);

        uint256 totalAssets = 0;
        uint256 strategiesLength = _strategies.length;

        for (uint256 i = 0; i < strategiesLength; i++) {
            currentAssets[i] = IStrategy(_strategies[i]).totalAssets();
            totalAssets += currentAssets[i];
        }
        totalUndeployed = 0;
        for (uint256 i = 0; i < strategiesLength; i++) {
            uint256 fractAmount = (amount * currentAssets[i]) / totalAssets;
+           if (fractAmount == 0) continue;
            totalUndeployed += IStrategy(_strategies[i]).undeploy(fractAmount);
        }
    }
```

**chefkenji (BakerFi) confirmed**

**[MrPotatoMagic (warden) commented](https://code4rena.com/audits/2024-12-bakerfi-invitational/submissions/F-33?commentParent=ZLEFPLMMYj5):**

> There is no DOS issue here. The `VAULT_MANAGER` can simply remove the strategy in that case. Having a strategy without any assets deposited means the strategy is unused and should be removed.

**[0xpiken (warden) commented](https://code4rena.com/audits/2024-12-bakerfi-invitational/submissions/F-33?commentParent=ZLEFPLMMYj5&commentChild=6pgbPkwiJ6b):**

> `VAULT_MANAGER` may add a new strategy with no assets allocated yet. All withdrawals since then will be DoS’ed.

**[MrPotatoMagic (warden) commented](https://code4rena.com/audits/2024-12-bakerfi-invitational/submissions/F-33?commentParent=ZLEFPLMMYj5&commentChild=F8f9HRn39XH):**

> `add a new strategy with no assets allocated yet.` - You’ve added it to allow users to deposit. I do not see this being any more than Low/Info finding. The availability of the protocol is only impacted till the time you deposit assets into the new strategy.

**[Dravee (judge) commented](https://code4rena.com/audits/2024-12-bakerfi-invitational/submissions/F-33?commentParent=ZLEFPLMMYj5&commentChild=uNXuVc4EdYC):**

> First and foremost: this was confirmed by the sponsor.
> Let’s now discuss about the severity.
>
> > The availability of the protocol is only impacted till the time you deposit assets into the new strategy.
>
> The protocol’s availability and functionality is indeed impacted unexpectedly.
> But there exist a workaround for this not to be permanent.
> Still, users’ assets can be affected quite badly (all withdrawals).
> This is an edge case, but it indeed qualifies as a Medium.

**[BakerFi mitigated](https://github.com/code-423n4/2025-01-bakerfi-mitigation?tab=readme-ov-file# findings-being-mitigated):**

> [PR-5](https://github.com/baker-fi/bakerfi-contracts/pull/5)

**Status:** Mitigation confirmed. Full details in reports from [0xlemon](https://code4rena.com/evaluate/2025-01-bakerfi-mitigation-review/findings/S-36) and [shaflow2](https://code4rena.com/evaluate/2025-01-bakerfi-mitigation-review/findings/S-12).

---

---
