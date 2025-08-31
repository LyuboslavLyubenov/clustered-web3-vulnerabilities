# Cluster -1400

**Rank:** #434  
**Count:** 9  

## Label
Front-running and improper state validation enable malicious actors to manipulate delegation states, block stake recovery, or evade penalties by exploiting transaction order and unchecked state transitions.

## Cluster Information
- **Total Findings:** 9

## Examples

### Example 1

**Auto Label:** Lack of state validation enables unauthorized asset manipulation, front-running, or denial of withdrawal, leading to financial distortion, liquidity lock-up, and system instability.  

**Original Text Preview:**

## 01. Relevant GitHub Links

* <https://github.com/Cyfrin/2024-12-alchemix/blob/82798f4891e41959eef866bd1d4cb44fc1e26439/src/StrategyMainnet.sol#L92C1-L113C6>
* <https://github.com/Cyfrin/2024-12-alchemix/blob/82798f4891e41959eef866bd1d4cb44fc1e26439/src/StrategyArb.sol#L71C1-L78C6>
* <https://github.com/Cyfrin/2024-12-alchemix/blob/82798f4891e41959eef866bd1d4cb44fc1e26439/src/StrategyOp.sol#L79C1-L89C6>

## 02. Summary

According to the [Tokenized Strategies documentation](https://docs.yearn.fi/developers/v3/strategy_writing_guide#_harvestandreport), once a strategy is shut down, no new assets should be deployed into the system. However, in `StrategyArb`, `StrategyMainnet`, and `StrategyOp` contracts, even after the strategy is shut down, a keeper can still call the claimAndSwap function. This function claims underlying assets and reintroduces them into the transmuter, effectively redeploying new assets post-shutdown, contrary to the intended behavior.

## 03. Vulnerability Details

The [Tokenized Strategies documentation](https://docs.yearn.fi/developers/v3/strategy_writing_guide#_harvestandreport) clearly states that upon shutdown, the strategy should prevent the deployment of any new assets. Yet, these contracts allow `claimAndSwap` calls after shutdown. While deposits and mints are blocked as intended, the `claimAndSwap` function lets a keeper claim underlying tokens and swap them back into the asset, depositing them into the transmuter. This unintentionally enables asset reinsertion into the system even after it is considered terminated.

```solidity
function claimAndSwap(
    uint256 _amountClaim, 
    uint256 _minOut, 
    uint256 _routeNumber
) external onlyKeepers {
    transmuter.claim(_amountClaim, address(this));
    uint256 balBefore = asset.balanceOf(address(this));
    require(_minOut > _amountClaim, "minOut too low");

    router.exchange(
        routes[_routeNumber],
        swapParams[_routeNumber],
        _amountClaim,
        _minOut,
        pools[_routeNumber],
        address(this)
    );        
    uint256 balAfter = asset.balanceOf(address(this));
    
    require((balAfter - balBefore) >= _minOut, "Slippage too high");
    transmuter.deposit(asset.balanceOf(address(this)), address(this));
}
```

In the canonical `_harvestAndReport` function shown in the documentation, safeguards exist to ensure no redeployment occurs once the strategy is shutdown. The current implementations in the provided strategies lack a similar check, allowing claim and swap operations that effectively circumvent the shutdown condition.

```solidity
function _harvestAndReport() internal override returns (uint256 _totalAssets) {
    // Only harvest and redeploy if the strategy is not shutdown.
    if(!TokenizedStrategy.isShutdown()) {
        // Claim all rewards and sell to asset.
        _claimAndSellRewards();
        
        // Check how much we can re-deploy into the yield source.
        uint256 toDeploy = Math.min(
            asset.balanceOf(address(this)), 
            availableDepositLimit(address(this))
        );
        
        // If greater than 0.
        if (toDeploy > 0) {
            // Deposit the sold amount back into the yield source.
            _deployFunds(toDeploy)
        }
    }
    
    // Return full balance no matter what.
    _totalAssets = yieldSource.balanceOf(address(this)) + asset.balanceOf(address(this));
}
```

## 04. Impact

1. Financial Manipulation: Continuous asset deployment post-shutdown can be leveraged to manipulate the system’s financial state, potentially leading to unexpected losses or imbalances.
2. System Instability: Persistent redeployment may cause inconsistencies in asset tracking, reporting, and overall strategy performance, making it difficult to maintain accurate records and recover funds if needed.
3. Trust and Reputation Damage: Stakeholders rely on the assurance that a shutdown effectively halts all operations. Failure to enforce this can lead to loss of confidence, adversely affecting user engagement and platform reputation.
4. Regulatory Concerns: Inconsistent behavior post-shutdown might attract regulatory scrutiny, especially if it leads to financial discrepancies or perceived negligence in safeguarding assets.

Given these factors, the vulnerability’s impact extends beyond mere operational inconsistencies, posing significant risks to financial integrity and stakeholder trust.

## 05. Proof of Concept

```solidity
pragma solidity ^0.8.18;

import "forge-std/console.sol";
import {Setup, ERC20, IStrategyInterface} from "./utils/Setup.sol";

import {IStrategyInterfaceVelo} from "../interfaces/IStrategyInterface.sol";
import {IStrategyInterfaceRamses} from "../interfaces/IStrategyInterface.sol";

import {IVeloRouter} from "../interfaces/IVelo.sol";
import {IRamsesRouter} from "../interfaces/IRamses.sol";

contract ShutdownTest is Setup {
    function setUp() public virtual override {
        super.setUp();
    }

    function test_poc_claimAndSwap_with_shutdown(uint256 _amount) public {
        vm.assume(_amount > minFuzzAmount && _amount < maxFuzzAmount);

        // Deposit into strategy
        mintAndDepositIntoStrategy(strategy, user, _amount);

        assertEq(strategy.totalAssets(), _amount, "!totalAssets");

        // Earn Interest
        skip(1 days);

        // Shutdown the strategy
        vm.prank(management);
        strategy.shutdownStrategy();

        // claimandswap
        console.log("===================================");
        console.log("Amount deposited:", _amount);
        console.log("Total Assets:", strategy.totalAssets());
        console.log("Claimable:", strategy.claimableBalance());
        console.log("Unexchanged Balance:", strategy.unexchangedBalance());
        console.log("Exchangable Balance:", transmuter.getExchangedBalance(address(strategy)));
        console.log("Total Unexchanged:", transmuter.totalUnexchanged());
        console.log("Total Buffered:", transmuter.totalBuffered());

        assertApproxEq(strategy.totalAssets(), _amount, _amount / 500);
        vm.roll(1);

        console.log("===================================");
        console.log("Total Assets in Strategy:", strategy.totalAssets());
        deployMockYieldToken();
        console.log("Deployed Mock Yield Token");
        addMockYieldToken();
        console.log("Added Mock Yield Token");
        depositToAlchemist(_amount);
        console.log("Deposited to Alchemist");
        airdropToMockYield(_amount / 2);
        console.log("Airdropped to Mock Yield");

        vm.prank(whale);
        asset.transfer(user2, _amount);

        vm.prank(user2);
        asset.approve(address(transmuter), _amount);
        
        vm.prank(user2);
        transmuter.deposit(_amount /2 , user2);

        vm.roll(1);
        harvestMockYield();

        vm.prank(address(transmuterKeeper));
        transmuterBuffer.exchange(address(underlying));

        skip(7 days);
        vm.roll(5);

        vm.prank(user2);
        transmuter.deposit(_amount /2 , user2);

        vm.prank(address(transmuterKeeper));
        transmuterBuffer.exchange(address(underlying));

        console.log("===================================");
        console.log("Total Assets in Strategy:", strategy.totalAssets());
        console.log("Skip 7 days");
        console.log("Claimable:", strategy.claimableBalance());
        console.log("Unexchanged Balance:", strategy.unexchangedBalance());
        console.log("Exchangable Balance:", transmuter.getExchangedBalance(address(strategy)));
        console.log("Total Unexchanged:", transmuter.totalUnexchanged());
        console.log("Total Buffered:", transmuter.totalBuffered());

        assertGt(strategy.claimableBalance(), 0, "!claimableBalance");
        assertEq(strategy.totalAssets(), _amount);
        uint256 claimable = strategy.claimableBalance();

        skip(1 seconds);
        vm.roll(1);

        vm.prank(keeper);

        if (block.chainid == 1) {
            IStrategyInterface(address(strategy)).claimAndSwap(
                claimable,
                claimable * 101 / 100,
                0
            );

        } else if (block.chainid == 10) {
            IVeloRouter.route[] memory veloRoute = new IVeloRouter.route[]();
            veloRoute[0] = IVeloRouter.route(address(underlying), address(asset), true, 0xF1046053aa5682b4F9a81b5481394DA16BE5FF5a);
            IStrategyInterfaceVelo(address(strategy)).claimAndSwap(claimable, claimable * 103 / 100, veloRoute);
        } else if (block.chainid == 42161) {
            IRamsesRouter.route[] memory  ramsesRoute = new IRamsesRouter.route[]();
            address eFrax = 0x178412e79c25968a32e89b11f63B33F733770c2A;
            ramsesRoute[0] = IRamsesRouter.route(address(underlying), eFrax, true);
            ramsesRoute[1] = IRamsesRouter.route(eFrax, address(asset), true);

            IStrategyInterfaceRamses(address(strategy)).claimAndSwap(claimable, claimable * 103 / 100, ramsesRoute);
        } else {
            revert("Chain ID not supported");
        }        

        console.log("===================================");
        console.log("Claimable:", strategy.claimableBalance());
        console.log("Unexchanged Balance:", strategy.unexchangedBalance());
        console.log("Exchangable Balance:", transmuter.getExchangedBalance(address(strategy)));
        console.log("Total Unexchanged:", transmuter.totalUnexchanged());
        console.log("Total Assets in Strategy:", strategy.totalAssets());
        console.log("Free Assets in Strategy:", asset.balanceOf(address(strategy)));
        console.log("Underlying in Strategy:", underlying.balanceOf(address(strategy)));

        vm.prank(keeper);
        (uint256 profit, uint256 loss) = strategy.report();

        assertEq(strategy.claimableBalance(), 0, "!claimableBalance");
        assertGt(strategy.totalAssets(), _amount, "!totalAssets");

    }
}
```

```Solidity
(hackenv) bshyuunn@hyuunn-MacBook-Air 2024-12-alchemix % forge test --mt test_poc_claimAndSwap_with_shutdown --fork-url <https://rpc.ankr.com/eth> -vvv 
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for src/test/Shutdown.t.sol:ShutdownTest
[PASS] test_poc_claimAndSwap_with_shutdown(uint256) (runs: 257, μ: 3958995, ~: 3939483)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 25.94s (22.20s CPU time)

Ran 1 test suite in 26.98s (25.94s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)

```

## 06. Tools Used

Manual Code Review and Foundry

## 07. Recommended Mitigation

```diff
	function claimAndSwap(
	    uint256 _amountClaim, 
	    uint256 _minOut, 
	    uint256 _routeNumber
	) external onlyKeepers {
	    transmuter.claim(_amountClaim, address(this));

++		if(!TokenizedStrategy.isShutdown()) {
				uint256 balBefore = asset.balanceOf(address(this));
   	    require(_minOut > _amountClaim, "minOut too low");

		    router.exchange(
		        routes[_routeNumber],
		        swapParams[_routeNumber],
		        _amountClaim,
		        _minOut,
		        pools[_routeNumber],
		        address(this)
		    );        
		    uint256 balAfter = asset.balanceOf(address(this));
		    
		    require((balAfter - balBefore) >= _minOut, "Slippage too high");
		    transmuter.deposit(asset.balanceOf(address(this)), address(this));
++    }
	}
```

---
### Example 2

**Auto Label:** Front-running and improper state validation enable malicious actors to manipulate delegation states, block stake recovery, or evade penalties by exploiting transaction order and unchecked state transitions.  

**Original Text Preview:**

## Security Report

## Severity
**Low Risk**

## Context
- `SonicStaking.sol#L341`
- `SonicStaking.sol#L505`

## Description
When calling `operatorInitiateClawBack()`, a user can front-run the call by calling `undelegate()` for the same `validatorId` and prevent `operatorInitiateClawBack()` from executing.

## Recommendation
Consider lowering the `amountAssets` to the amount that is available, for example in the following way:

```solidity
function operatorInitiateClawBack(/*...*/, uint256 amountAssets) /*...*/ {
    // ...
    uint256 amountDelegated = SFC.getStake(address(this), validatorId);
    + if (amountAssets > amountDelegated) {
    +     amountAssets = amountDelegated;
    + }
    // ...
    - require(amountAssets <= amountDelegated, UndelegateAmountExceedsDelegated(validatorId));
    // ...
}
```

## Status
- **Beethoven:** Addressed in PR 61 and PR 62.
- **Spearbit:** Fixed.

---
### Example 3

**Auto Label:** Lack of state validation enables unauthorized asset manipulation, front-running, or denial of withdrawal, leading to financial distortion, liquidity lock-up, and system instability.  

**Original Text Preview:**

## Severity: Low Risk

## Context
EulerEarnVault.sol#L507

## Description
The `_withdraw` function iterates over strategies in the withdrawal queue order and withdraws from the strategies in case more withdrawable assets are needed. This can be exploited by a griefer to off-balance the Earn vault strategy allocation, for example, leaving all assets in the last strategy of the withdrawal queue, affecting the APR of the Earn vault.

## Example
Imagine a scenario where $10M is misallocated evenly across the cash reserve and 4 strategies ($2M in each "strategy"). A user performs the following actions in a single transaction:
- Flashloan $40M.
- Deposit $40M.
- Rebalance(AllStrategies). Each strategy now holds $10M.
- Withdraw $40M. The cash reserve only has $10M, meaning the other $30M will be withdrawn from the first three strategies in the withdrawal queue.
- The final result is that the entire $10M initial total assets are in the last strategy.

## Recommendation
A rebalance should be performed periodically.

## Euler
Fixed in commit b5b19d67.

## Spearbit
Verified.

---
