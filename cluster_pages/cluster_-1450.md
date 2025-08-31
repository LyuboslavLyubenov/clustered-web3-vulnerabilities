# Cluster -1450

**Rank:** #166  
**Count:** 77  

## Label
Inaccurate state updates due to improper edge case handling, leading to fund locking, incorrect matching, or erroneous token transfers and enabling denial-of-service or financial loss.

## Cluster Information
- **Total Findings:** 77

## Examples

### Example 1

**Auto Label:** Inadequate validation and flawed state transitions lead to incorrect asset calculations, unauthorized access, or unfair value distribution during withdrawals.  

**Original Text Preview:**

**Description:** After transitioning to the yield phase, redemptions of both pUSDe and yUSDe are processed by `pUSDeVault::_withdraw` such that they are both paid out in sUSDe. This is achieved by computing the sUSDe balance corresponding to the required USDe amount by calling its `previewWithdraw()` function:

```solidity
    function _withdraw(address caller, address receiver, address owner, uint256 assets, uint256 shares) internal override {

            if (PreDepositPhase.YieldPhase == currentPhase) {
                // sUSDeAssets = sUSDeAssets + user_yield_sUSDe
@>              assets += previewYield(caller, shares);

@>              uint sUSDeAssets = sUSDe.previewWithdraw(assets); // @audit - this rounds up because sUSDe requires the amount of sUSDe burned to receive assets amount of USDe to round up, but below we are transferring this rounded value out to the receiver which actually rounds against the protocol/yUSDe depositors!

                _withdraw(
                    address(sUSDe),
                    caller,
                    receiver,
                    owner,
                    assets, // @audit - this should not include the yield, since it is decremented from depositedBase
                    sUSDeAssets,
                    shares
                );
                return;
            }
        ...
    }
```

The issue with this is that `previewWithdraw()` returns the required sUSDe balance that must be burned to receive the specified USDe amount and so rounds up accordingly; however, here this rounded sUSDe amount is being transferred out of the protocol. This means that the redemption actually rounds in favour of the receiver and against the protocol/yUSDe depositors.

**Impact:** Value can leak from the system in favour of pUSDe redemptions at the expense of other yUSDe depositors.

**Proof of Concept:** Note that the following test will revert due to underflow when attempting to determine the fully redeemed amounts unless the mitigation from C-01 is applied:

```solidity
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {IERC4626} from "@openzeppelin/contracts/interfaces/IERC4626.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

import {MockUSDe} from "../contracts/test/MockUSDe.sol";
import {MockStakedUSDe} from "../contracts/test/MockStakedUSDe.sol";
import {MockERC4626} from "../contracts/test/MockERC4626.sol";

import {pUSDeVault} from "../contracts/predeposit/pUSDeVault.sol";
import {yUSDeVault} from "../contracts/predeposit/yUSDeVault.sol";

import {console2} from "forge-std/console2.sol";

contract RoundingTest is Test {
    uint256 constant MIN_SHARES = 0.1 ether;

    MockUSDe public USDe;
    MockStakedUSDe public sUSDe;
    pUSDeVault public pUSDe;
    yUSDeVault public yUSDe;

    address account;

    address alice = makeAddr("alice");
    address bob = makeAddr("bob");

    function setUp() public {
        address owner = msg.sender;

        USDe = new MockUSDe();
        sUSDe = new MockStakedUSDe(USDe, owner, owner);

        pUSDe = pUSDeVault(
            address(
                new ERC1967Proxy(
                    address(new pUSDeVault()),
                    abi.encodeWithSelector(pUSDeVault.initialize.selector, owner, USDe, sUSDe)
                )
            )
        );

        yUSDe = yUSDeVault(
            address(
                new ERC1967Proxy(
                    address(new yUSDeVault()),
                    abi.encodeWithSelector(yUSDeVault.initialize.selector, owner, USDe, sUSDe, pUSDe)
                )
            )
        );

        vm.startPrank(owner);
        pUSDe.setDepositsEnabled(true);
        pUSDe.setWithdrawalsEnabled(true);
        pUSDe.updateYUSDeVault(address(yUSDe));

        // deposit USDe and burn minimum shares to avoid reverting on redemption
        uint256 initialUSDeAmount = pUSDe.previewMint(MIN_SHARES);
        USDe.mint(owner, initialUSDeAmount);
        USDe.approve(address(pUSDe), initialUSDeAmount);
        pUSDe.mint(MIN_SHARES, address(0xdead));
        vm.stopPrank();

        if (pUSDe.balanceOf(address(0xdead)) != MIN_SHARES) {
            revert("address(0xdead) should have MIN_SHARES shares of pUSDe");
        }
    }

    function test_rounding() public {
        uint256 userDeposit = 100 ether;

        // fund users
        USDe.mint(alice, userDeposit);
        USDe.mint(bob, userDeposit);

        // alice deposits into pUSDe
        vm.startPrank(alice);
        USDe.approve(address(pUSDe), userDeposit);
        uint256 aliceShares_pUSDe = pUSDe.deposit(userDeposit, alice);
        vm.stopPrank();

        // bob deposits into pUSDe
        vm.startPrank(bob);
        USDe.approve(address(pUSDe), userDeposit);
        uint256 bobShares_pUSDe = pUSDe.deposit(userDeposit, bob);
        vm.stopPrank();

        // setup assertions
        assertEq(pUSDe.balanceOf(alice), aliceShares_pUSDe, "Alice should have shares equal to her deposit");
        assertEq(pUSDe.balanceOf(bob), bobShares_pUSDe, "Bob should have shares equal to his deposit");

        {
            // phase change
            account = msg.sender;
            uint256 initialAdminTransferAmount = 1e6;
            vm.startPrank(account);
            USDe.mint(account, initialAdminTransferAmount);
            USDe.approve(address(pUSDe), initialAdminTransferAmount);
            pUSDe.deposit(initialAdminTransferAmount, address(yUSDe));
            pUSDe.startYieldPhase();
            yUSDe.setDepositsEnabled(true);
            yUSDe.setWithdrawalsEnabled(true);
            vm.stopPrank();
        }

        // bob deposits into yUSDe
        vm.startPrank(bob);
        pUSDe.approve(address(yUSDe), bobShares_pUSDe);
        uint256 bobShares_yUSDe = yUSDe.deposit(bobShares_pUSDe, bob);
        vm.stopPrank();

        // simulate sUSDe yield transfer
        uint256 sUSDeYieldAmount = 1_000 ether;
        USDe.mint(address(sUSDe), sUSDeYieldAmount);

        // alice redeems from pUSDe
        uint256 aliceBalanceBefore_sUSDe = sUSDe.balanceOf(alice);
        vm.prank(alice);
        uint256 aliceRedeemed_USDe_reported = pUSDe.redeem(aliceShares_pUSDe, alice, alice);
        uint256 aliceRedeemed_sUSDe = sUSDe.balanceOf(alice) - aliceBalanceBefore_sUSDe;
        uint256 aliceRedeemed_USDe_actual = sUSDe.previewRedeem(aliceRedeemed_sUSDe);

        // bob redeems from yUSDe
        uint256 bobBalanceBefore_sUSDe = sUSDe.balanceOf(bob);
        vm.prank(bob);
        uint256 bobRedeemed_pUSDe_reported = yUSDe.redeem(bobShares_yUSDe, bob, bob);
        uint256 bobRedeemed_sUSDe = sUSDe.balanceOf(bob) - bobBalanceBefore_sUSDe;
        uint256 bobRedeemed_USDe = sUSDe.previewRedeem(bobRedeemed_sUSDe);

        console2.log("Alice redeemed sUSDe: %s", aliceRedeemed_sUSDe);
        console2.log("Alice redeemed USDe (reported): %s", aliceRedeemed_USDe_reported);
        console2.log("Alice redeemed USDe (actual): %s", aliceRedeemed_USDe_actual);

        console2.log("Bob redeemed pUSDe (reported): %s", bobRedeemed_pUSDe_reported);
        console2.log("Bob redeemed pUSDe (actual): %s", bobShares_pUSDe);
        console2.log("Bob redeemed sUSDe: %s", bobRedeemed_sUSDe);
        console2.log("Bob redeemed USDe: %s", bobRedeemed_USDe);

        // post-redemption assertions
        assertEq(
            aliceRedeemed_USDe_reported,
            aliceRedeemed_USDe_actual,
            "Alice's reported and actual USDe redemption amounts should match"
        );

        assertGe(
            bobRedeemed_pUSDe_reported,
            bobShares_pUSDe,
            "Bob should redeem at least the same amount of pUSDe as his original deposit"
        );

        assertGe(
            bobRedeemed_USDe, userDeposit, "Bob should redeem at least the same amount of USDe as his initial deposit"
        );

        assertLe(
            aliceRedeemed_USDe_actual,
            userDeposit,
            "Alice should redeem no more than the same amount of USDe as her initial deposit"
        );
    }
}
```

The following Echidna optimization test can also be run to maximise this discrepancy:

```solidity
// SPDX-License-Identifier: GPL-2.0
pragma solidity ^0.8.0;

import {BaseSetup} from "@chimera/BaseSetup.sol";
import {CryticAsserts} from "@chimera/CryticAsserts.sol";
import {vm} from "@chimera/Hevm.sol";

import {pUSDeVault} from "contracts/predeposit/pUSDeVault.sol";
import {yUSDeVault} from "contracts/predeposit/yUSDeVault.sol";
import {MockUSDe} from "contracts/test/MockUSDe.sol";
import {MockStakedUSDe} from "contracts/test/MockStakedUSDe.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

// echidna . --contract CryticRoundingTester --config echidna_rounding.yaml --format text --workers 16 --test-limit 1000000
contract CryticRoundingTester is BaseSetup, CryticAsserts {
    uint256 constant MIN_SHARES = 0.1 ether;

    MockUSDe USDe;
    MockStakedUSDe sUSDe;
    pUSDeVault pUSDe;
    yUSDeVault yUSDe;

    address owner;
    address alice = address(uint160(uint256(keccak256(abi.encodePacked("alice")))));
    address bob = address(uint160(uint256(keccak256(abi.encodePacked("bob")))));
    uint256 severity;

    constructor() payable {
        setup();
    }

    function setup() internal virtual override {
        owner = msg.sender;

        USDe = new MockUSDe();
        sUSDe = new MockStakedUSDe(USDe, owner, owner);

        pUSDe = pUSDeVault(
            address(
                new ERC1967Proxy(
                    address(new pUSDeVault()),
                    abi.encodeWithSelector(pUSDeVault.initialize.selector, owner, USDe, sUSDe)
                )
            )
        );

        yUSDe = yUSDeVault(
            address(
                new ERC1967Proxy(
                    address(new yUSDeVault()),
                    abi.encodeWithSelector(yUSDeVault.initialize.selector, owner, USDe, sUSDe, pUSDe)
                )
            )
        );

        vm.startPrank(owner);
        pUSDe.setDepositsEnabled(true);
        pUSDe.setWithdrawalsEnabled(true);
        pUSDe.updateYUSDeVault(address(yUSDe));

        // deposit USDe and burn minimum shares to avoid reverting on redemption
        uint256 initialUSDeAmount = pUSDe.previewMint(MIN_SHARES);
        USDe.mint(owner, initialUSDeAmount);
        USDe.approve(address(pUSDe), initialUSDeAmount);
        pUSDe.mint(MIN_SHARES, address(0xdead));
        vm.stopPrank();

        if (pUSDe.balanceOf(address(0xdead)) != MIN_SHARES) {
            revert("address(0xdead) should have MIN_SHARES shares of pUSDe");
        }
    }

    function target(uint256 aliceDeposit, uint256 bobDeposit, uint256 sUSDeYieldAmount) public {
        aliceDeposit = between(aliceDeposit, 1, 100_000 ether);
        bobDeposit = between(bobDeposit, 1, 100_000 ether);
        sUSDeYieldAmount = between(sUSDeYieldAmount, 1, 500_000 ether);
        precondition(aliceDeposit <= 100_000 ether);
        precondition(bobDeposit <= 100_000 ether);
        precondition(sUSDeYieldAmount <= 500_000 ether);

        // fund users
        USDe.mint(alice, aliceDeposit);
        USDe.mint(bob, bobDeposit);

        // alice deposits into pUSDe
        vm.startPrank(alice);
        USDe.approve(address(pUSDe), aliceDeposit);
        uint256 aliceShares_pUSDe = pUSDe.deposit(aliceDeposit, alice);
        vm.stopPrank();

        // bob deposits into pUSDe
        vm.startPrank(bob);
        USDe.approve(address(pUSDe), bobDeposit);
        uint256 bobShares_pUSDe = pUSDe.deposit(bobDeposit, bob);
        vm.stopPrank();

        // setup assertions
        eq(pUSDe.balanceOf(alice), aliceShares_pUSDe, "Alice should have shares equal to her deposit");
        eq(pUSDe.balanceOf(bob), bobShares_pUSDe, "Bob should have shares equal to his deposit");

        {
            // phase change
            uint256 initialAdminTransferAmount = 1e6;
            vm.startPrank(owner);
            USDe.mint(owner, initialAdminTransferAmount);
            USDe.approve(address(pUSDe), initialAdminTransferAmount);
            pUSDe.deposit(initialAdminTransferAmount, address(yUSDe));
            pUSDe.startYieldPhase();
            yUSDe.setDepositsEnabled(true);
            yUSDe.setWithdrawalsEnabled(true);
            vm.stopPrank();
        }

        // bob deposits into yUSDe
        vm.startPrank(bob);
        pUSDe.approve(address(yUSDe), bobShares_pUSDe);
        uint256 bobShares_yUSDe = yUSDe.deposit(bobShares_pUSDe, bob);
        vm.stopPrank();

        // simulate sUSDe yield transfer
        USDe.mint(address(sUSDe), sUSDeYieldAmount);

        // alice redeems from pUSDe
        uint256 aliceBalanceBefore_sUSDe = sUSDe.balanceOf(alice);
        vm.prank(alice);
        uint256 aliceRedeemed_USDe_reported = pUSDe.redeem(aliceShares_pUSDe, alice, alice);
        uint256 aliceRedeemed_sUSDe = sUSDe.balanceOf(alice) - aliceBalanceBefore_sUSDe;
        uint256 aliceRedeemed_USDe_actual = sUSDe.previewRedeem(aliceRedeemed_sUSDe);

        // bob redeems from yUSDe
        uint256 bobBalanceBefore_sUSDe = sUSDe.balanceOf(bob);
        vm.prank(bob);
        uint256 bobRedeemed_pUSDe_reported = yUSDe.redeem(bobShares_yUSDe, bob, bob);
        uint256 bobRedeemed_sUSDe = sUSDe.balanceOf(bob) - bobBalanceBefore_sUSDe;
        uint256 bobRedeemed_USDe = sUSDe.previewRedeem(bobRedeemed_sUSDe);

        // optimize
        if (aliceRedeemed_USDe_actual > aliceDeposit) {
            uint256 diff = aliceRedeemed_USDe_actual - aliceDeposit;
            if (diff > severity) {
                severity = diff;
            }
        }
    }

    function echidna_opt_severity() public view returns (uint256) {
        return severity;
    }
}
```

Config:
```yaml
testMode: "optimization"
prefix: "echidna_"
coverage: true
corpusDir: "echidna_rounding"
balanceAddr: 0x1043561a8829300000
balanceContract: 0x1043561a8829300000
filterFunctions: []
cryticArgs: ["--foundry-compile-all"]
deployer: "0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496"
contractAddr: "0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496"
shrinkLimit: 100000
```

Output:
```bash
echidna_opt_severity: max value: 444330
```

**Recommended Mitigation:** Rather than calling `previewWithdraw()` which rounds up, call `convertToShares()` which rounds down:

```solidity
function previewWithdraw(uint256 assets) public view virtual override returns (uint256) {
    return _convertToShares(assets, Math.Rounding.Up);
}

function convertToShares(uint256 assets) public view virtual override returns (uint256) {
    return _convertToShares(assets, Math.Rounding.Down);
}
```

**Strata:** Fixed in commit [59fcf23](https://github.com/Strata-Money/contracts/commit/59fcf239a9089d14f02621a7f692bcda6c85690e).

**Cyfrin:** Verified. The sUSDe to transfer out to the receiver is now calculated using `convertToShares()` which rounds down.

\clearpage

---
### Example 2

**Auto Label:** Failure to validate or simulate transaction outcomes before execution, leading to unpredictable asset changes, user deception, or unbounded token supply.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-04-zetachain-cross-chain-judging/issues/323 

This issue has been acknowledged by the team but won't be fixed at this time.

## Found by 
berndartmueller

## Summary

ZETA tokens minted to the fungible module account are not burned/reverted if the `onReceive()` ZEVM call reverts, which keeps increasing the total supply of ZETA tokens. This can lead to a denial of service (DoS) for ZETA deposits on ZetaChain, as the cap of 1.85 billion ZETA tokens is reached.

## Root Cause

In `CallOnReceiveZevmConnector()`, ZETA tokens are first [minted to the fungible module](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/x/fungible/keeper/evm.go#L380) by calling `DepositCoinsToFungibleModule()`, so that the subsequent EVM call has sufficient [native tokens (ZETA) to attach](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/x/fungible/keeper/evm.go#L390).

If we follow the code path up,

- [`InitiateOutbound()`](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/x/crosschain/keeper/cctx_gateway_zevm.go#L26-L46)
  - [`HandleEVMDeposit()`](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/x/crosschain/keeper/evm_deposit.go#L27-L66)
    - [`ZETADepositAndCallContract()`](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/x/fungible/keeper/zevm_msg_passing.go#L17-L46)
      - `CallOnReceiveZevmConnector()`

we notice that in `InitiateOutbound()` a [temporary context `tmpCtx` is used and passed down](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/x/crosschain/keeper/cctx_gateway_zevm.go#L45-L46) to the chain of function calls.

[This temporary context is committed](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/x/crosschain/keeper/cctx_gateway_zevm.go#L60), i.e., the state is written, if the CCTX's outbound is either successfully mined or if the CCTX will get reverted:

```go
58: newCCTXStatus = c.crosschainKeeper.ValidateOutboundZEVM(ctx, config.CCTX, err, isContractReverted)
59: if newCCTXStatus == types.CctxStatus_OutboundMined || newCCTXStatus == types.CctxStatus_PendingRevert {
60: 	commit()
61: }
```

However, if the [`onReceive()` EVM call in `CallOnReceiveZevmConnector()`](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/x/fungible/keeper/evm.go#L385-L401) reverts (e.g., the target contract reverts), the ZETA tokens remain minted to the fungible module while reverting the CCTX and thus refunding the ZETA tokens to the sender on the external chain (e.g., Ethereum).

As a result, ZETA tokens are minted out of thin air to the fungible module account, which increases the total supply of ZETA tokens. This is a problem as the ZETA token has a [**cap of 1.85 billion ZETA tokens**](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/x/fungible/keeper/zeta.go#L16), checked by [`validateZetaSupply()`](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/x/fungible/keeper/zeta.go#L45-L58). If the cap is reached, [minting ZETA tokens will fail](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/x/fungible/keeper/zeta.go#L18-L43) and thus prevents ZETA deposits on ZetaChain. This is effectively a DoS.

Same also in case of `processFailedZETAOutboundOnZEVM()` (which internally calls `CallOnRevertZevmConnector()` that also mints ZETA tokens to the fungible module without a temporary context). However, due to [aborting the CCTX in case of the failed revert](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/x/crosschain/keeper/cctx_orchestrator_validate_outbound.go#L254-L261), without refunding the funds immediately, it does not invite for an attack. Reclaiming the funds would require an admin to refund them via the `MsgRefundAbortedCCTX` message.

## Internal Pre-conditions

None

## External Pre-conditions

None

## Attack Path

Given ZETA at the time of the review is valued at ~$0.25 USD. And an attacker has a ZEVM contract deployed on ZetaChain that reverts in `onReceive()`.

An attacker can purposefully trigger the 1.85B ZETA supply cap by repeating the following steps:

1. On Ethereum, initiate a ZETA token inbound via `ZetaConnectorEth.send()` to ZetaChain with 10M ZETA tokens, worth $2.5M USD.
2. CCTX inbound errors on ZetaChain when executing the `onReceive()` call on the target contract. The 10M ZETA tokens have been minted to the fungible module account.
3. A revert outbound is created to refund the 10M ZETA tokens to the sender on Ethereum. 4. Attacker receives the 10M ZETA tokens back on Ethereum.

Repeat 185 times until the cap is reached.

The gas fees are negligible. Also, the capital is returned to the attacker after each iteration, reducing the risk of failure.

## Impact

ZETA deposits and message passing via ZetaChain are halted. A network upgrade is required to burn the fungible module account ZETA tokens.

## PoC

Add the following test case to [`node/x/fungible/keeper/evm_test.go`](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/x/fungible/keeper/evm_test.go) and run with

```bash
go test -timeout 30s -run ^TestKeeper_CallOnReceiveZevmConnector$/^AUDIT_should_error_in_contract_call_reverts_but_ZETA_is_minted_to_fungible_module_account$ github.com/zeta-chain/node/x/fungible/keeper
```

```diff
diff --git a/node/x/fungible/keeper/evm_test.go b/node/x/fungible/keeper/evm_test.go
--- node/x/fungible/keeper/evm_test.go
+++ node/x/fungible/keeper/evm_test.go
@@ -19,8 +19,9 @@
 	"github.com/zeta-chain/protocol-contracts/pkg/systemcontract.sol"
 	"github.com/zeta-chain/protocol-contracts/pkg/wzeta.sol"
 	"github.com/zeta-chain/protocol-contracts/pkg/zrc20.sol"

+	zetaconfig "github.com/zeta-chain/node/cmd/zetacored/config"
 	"github.com/zeta-chain/node/e2e/contracts/dapp"
 	"github.com/zeta-chain/node/e2e/contracts/dappreverter"
 	"github.com/zeta-chain/node/e2e/contracts/example"
 	"github.com/zeta-chain/node/e2e/contracts/reverter"
@@ -1562,8 +1563,34 @@
 			dAppContract,
 			big.NewInt(45), []byte("message"), [32]byte{})
 		require.ErrorContains(t, err, "execution reverted")
 	})
+
+	t.Run("AUDIT should error in contract call reverts but ZETA is minted to fungible module account", func(t *testing.T) {
+		// add import "zetaconfig "github.com/zeta-chain/node/cmd/zetacored/config""
+		k, ctx, sdkk, _ := keepertest.FungibleKeeper(t)
+		_ = k.GetAuthKeeper().GetModuleAccount(ctx, types.ModuleName)
+
+		deploySystemContracts(t, ctx, k, sdkk.EvmKeeper)
+		dAppContract, err := k.DeployContract(ctx, dappreverter.DappReverterMetaData)
+		require.NoError(t, err)
+		assertContractDeployment(t, sdkk.EvmKeeper, ctx, dAppContract)
+
+		// check fungible module address cosmos bank coin ZETA balance before the call
+		beforeBalance := sdkk.BankKeeper.GetBalance(ctx, types.ModuleAddress, zetaconfig.BaseDenom)
+		require.Equal(t, "0", beforeBalance.Amount.String())
+
+		_, err = k.CallOnReceiveZevmConnector(ctx,
+			sample.EthAddress().Bytes(),
+			big.NewInt(1),
+			dAppContract,
+			big.NewInt(45), []byte("message"), [32]byte{})
+		require.ErrorContains(t, err, "execution reverted")
+
+		// check fungible module address cosmos bank coin ZETA balance after the call
+		afterBalance := sdkk.BankKeeper.GetBalance(ctx, types.ModuleAddress, zetaconfig.BaseDenom)
+		require.Equal(t, "45", afterBalance.Amount.String()) // 45 ZETA is minted to fungible module account
+	})
 }

 func TestKeeper_CallOnRevertZevmConnector(t *testing.T) {
 	t.Run("should call on revert on connector which calls onZetaRevert on sample DAPP", func(t *testing.T) {
```

The test demonstrates that ZETA tokens are minted to the fungible module account even if the contract call reverts.

## Mitigation

Consider using another temporary context for both the ZETA token mint and the `onReceive()` EVM call to ensure atomicity. Same for the `processFailedZETAOutboundOnZEVM()` function.

---
### Example 3

**Auto Label:** **Insufficient input or state validation leads to asset loss, inconsistent state, or unauthorized operations through unchecked data or token mismatches.**  

**Original Text Preview:**

Whenever users call `openPosition()` or `closePosition()` they pass the `path` variable which is used to perform swaps. The issue is that the code doesn't verify the value of `path` to make sure that the first and last addresses in the path list are equal to the tokens that are going to be swapped. In some cases if the user specifies a wrong value for the last address in the path then the code would receive the wrong tokens and those tokens would stay in the contract address and the user would lose funds. It's better to verify the `path` parameter's value to avoid such edge cases.

---
