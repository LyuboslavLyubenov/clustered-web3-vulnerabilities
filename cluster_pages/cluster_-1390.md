# Cluster -1390

**Rank:** #293  
**Count:** 26  

## Label
Inadequate validation and flawed state transitions lead to incorrect asset calculations, unauthorized access, or unfair value distribution during withdrawals.

## Cluster Information
- **Total Findings:** 26

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

**Auto Label:** Insufficient validation of ownership and state conditions leads to unfair advantages, unauthorized state changes, and exploitable game mechanics.  

**Original Text Preview:**

Take a look at the `OPFaultVerifier` implementation:

<https://github.com/code-423n4/2024-12-unruggable/blob/8c8cdc170741d4c93287302acd8bca8ff3d625a2/contracts/op/OPFaultVerifier.sol# L67-L91>
```

interface IOptimismPortal {
    function disputeGameFactory() external view returns (IDisputeGameFactory);
    function respectedGameType() external view returns (GameType);
    // we don't care if the root was blacklisted, this only applies to withdrawals
    //function disputeGameBlacklist(IDisputeGame game) external view returns (bool);
}
..snip
function getStorageValues(
    bytes memory context,
    GatewayRequest memory req,
    bytes memory proof
) external view returns (bytes[] memory, uint8 exitCode) {
    uint256 gameIndex1 = abi.decode(context, (uint256));
    GatewayProof memory p = abi.decode(proof, (GatewayProof));
    (, , IDisputeGame gameProxy, uint256 blockNumber) = _gameFinder.gameAtIndex(
        _portal,
        _minAgeSec,
        _gameTypeBitMask,
        p.gameIndex
    );
    require(blockNumber != 0, "OPFault: invalid game");
    // @audit No blacklist check here
    // ...
    return GatewayVM.evalRequest(req, ProofSequence(0, p.outputRootProof.stateRoot, p.proofs, p.order, _hooks));
}
```

From the [documentation](https://github.com/code-423n4/2024-12-unruggable/blob/8c8cdc170741d4c93287302acd8bca8ff3d625a2/contracts/op/OPFaultVerifier.sol# L12-L13), the Optimism Fault proof verifier assumes that blacklisted games **only matters for withdrawals**, where as this is one of the reason why the blacklisted logic is implemented (i.e., blockage of withdrawals), this is not the only reason.

To put this in retrospect, The optimism portal allows the `GUARDIAN` to **permanently blacklist** a game, when the game resolves incorrectly, that’s to show this is no case to be trusted to be used in our verification logic.

This can be seen from the [Optimism Portal](https://github.com/code-423n4/2024-07-optimism/blob/2e11e15e7a9a86f90de090ebf9e3516279e30897/packages/contracts-bedrock/src/L1/OptimismPortal2.sol# L1-L3) itself.

See the implementation of the portal from the previous code4rena audit:

First, the blacklist is implemented as a simple mapping in the OptimismPortal2 contract:

[OptimismPortal2.sol# L127-L130](https://github.com/code-423n4/2024-07-optimism/blob/2e11e15e7a9a86f90de090ebf9e3516279e30897/packages/contracts-bedrock/src/L1/OptimismPortal2.sol# L127-L130)
```

/// @notice A mapping of dispute game addresses to whether or not they are blacklisted.
mapping(IDisputeGame => bool) public disputeGameBlacklist;
```

Only the guardian can blacklist dispute games and this is done only in the case where the game is **resolving incorrectly**:

[OptimismPortal2.sol# L439-L445](https://github.com/code-423n4/2024-07-optimism/blob/2e11e15e7a9a86f90de090ebf9e3516279e30897/packages/contracts-bedrock/src/L1/OptimismPortal2.sol# L439-L445)
```

/// @notice Blacklists a dispute game. Should only be used in the event that a dispute game resolves incorrectly.
/// @param _disputeGame Dispute game to blacklist.
function blacklistDisputeGame(IDisputeGame _disputeGame) external {
    if (msg.sender != guardian()) revert Unauthorized();
    disputeGameBlacklist[_disputeGame] = true;
    emit DisputeGameBlacklisted(_disputeGame);
}
```

Which then brings us to the fact that if a game is blacklisted, then it is a wrong premise to be used for verification and we should check for this in the Verifier.

### Impact

High. While the primary and most visible impact of blacklisting a game is blocking withdrawals, the blacklisting mechanism actually invalidates a dispute game entirely since they are resolving incorrectly which then means we shouldn’t use this in our OP Verifiers, otherwise we would be accepting an incorrectly resolving game.

### Recommended Mitigation Steps

Ensure the game being verified is not blacklisted.

**[thomasclowes (Unruggable) acknowledged and commented](https://code4rena.com/audits/2024-12-unruggable-invitational/submissions/F-13?commentParent=MAmKS4gXfP3):**

> The Optimism documentation is lacking, and this rollup tech is nascent in nature. From our discussions with the OP team the blacklisting is purely utilised for withdrawals. This is corroborated within the smart contracts.
>
> The overhead of checking against the blacklist is significant, and it is unclear **how** to verify against it in this context noting that the system wasn’t built with this use case in mind.
>
> We acknowledge this, but need to converse with the OP team directly as they iterate on their tech moving forward to be able to utilize this in any meaningful way. It was an intentional ommission.

**[Picodes (judge) commented](https://code4rena.com/audits/2024-12-unruggable-invitational/submissions/F-13?commentParent=MAmKS4gXfP3&commentChild=PWg9DFHRea9):**

> Downgrading to Low for “unclear documentation” following this remark.

**[Bauchibred (warden) commented](https://code4rena.com/audits/2024-12-unruggable-invitational/submissions/F-13?commentParent=2pgDbh6jEBy):**

> @Picodes, thanks for judging, but I can’t seem to understand the POV of the sponsors and take it at face value.
>
> I don’t see how we should be allowing a blacklisted game in Unruggable’s OPVerifier, considering **blacklisting a game** on Optimism is a **completely irreversible process**. This is only done once the game is concluded to have been resolved incorrectly by the Guardian.
>
> Does this block withdrawals? 100%, but the game is never permanently blacklisted if it *didn’t resolve incorrectly* and once done, it can not be un-done. In my opinion, it seems counterintuitive for sponsors to rely on the fact that it’s purely for withdrawals and have it here in the OPVerifier.

**[Picodes (judge) commented](https://code4rena.com/audits/2024-12-unruggable-invitational/submissions/F-13?commentParent=2pgDbh6jEBy&commentChild=WbpdwtoR72G):**

> @bauchibred - indeed digging into this this isn’t about Unruggable’s documentation, but about Optimism. However, I’m seeing the conditions required for this to happen. I believe Medium severity is more appropriate (“Assets not at direct risk, but the function of the protocol or its availability could be impacted, or leak value with a hypothetical attack path with stated assumptions, but external requirements.“)

**[thomasclowes (Unruggable) commented](https://code4rena.com/audits/2024-12-unruggable-invitational/submissions/F-13?commentParent=2pgDbh6jEBy&commentChild=wn5hW6u68vu):**

> The argument here is that the blacklist is not implemented as a method of verifying data integrity when reading data from OP stack chains. It is purely implemented in the context of asset withdrawals as outlined by Optimism. That is to say, Optimism would not blacklist a game to invalidate data read integrity..

**[Picodes (judge) commented](https://code4rena.com/audits/2024-12-unruggable-invitational/submissions/F-13?commentParent=2pgDbh6jEBy&commentChild=Y2HnKdsLvXQ):**

> @thomasclowes - I get the argument; but to me, the argument is not if they “would not” blacklist this game, but if they “can” do it or not, isn’t it? As they can, your reasoning stands but this is a valid Medium following C4 rules, that you’re free to acknowledge.

**[thomasclowes (Unruggable) commented](https://code4rena.com/audits/2024-12-unruggable-invitational/submissions/F-13?commentParent=2pgDbh6jEBy&commentChild=kKZBPSJNHH8):**

> They ‘can’ blacklist the game, but it is a mechanism that is not used in the context of data verification (this use case).

**[Picodes (judge) commented](https://code4rena.com/audits/2024-12-unruggable-invitational/submissions/F-13?commentParent=2pgDbh6jEBy&commentChild=yqt36ydpdDf):**

> @thomasclowes - is it explicitly outlined somewhere or is this from your conversation with the team?

**[thomasclowes (Unruggable) commented](https://code4rena.com/audits/2024-12-unruggable-invitational/submissions/F-13?commentParent=2pgDbh6jEBy&commentChild=uV2QcPKoCRS):**

> Conversations with the Optimism team, on their developers Discord. They stated that our use case for verifying state integrity is not what the blacklist is designed for and linked to the following [documentation](https://specs.optimism.io/fault-proof/stage-one/bridge-integration.html# blacklisting-disputegames).
>
> > “Blacklisting a dispute game means that no withdrawals proven against it will be allowed to finalize (per the “Dispute Game Blacklist” invariant), and they must re-prove against a new dispute game that resolves correctly.”

**Unruggable mitigated:**

> Mitigation [here](https://github.com/unruggable-labs/unruggable-gateways/commit/92c9fbb36dee3a11e0cff1096ce4958ac1491ae6).

**Status:** Mitigation confirmed. Resolved by adding finalized mode validation to check that the selected game is finalized.

---

---
### Example 3

**Auto Label:** Insufficient validation of ownership and state conditions leads to unfair advantages, unauthorized state changes, and exploitable game mechanics.  

**Original Text Preview:**

## Context
(No context files were provided by the reviewer)

## Description
The contract allows players to request spins and deposit funds without first registering. This creates a fundamental inconsistency, as the system assumes that only registered players who meet the age requirement should be able to participate. Without enforcing a registration check:
- Players can bypass the registration process and interact with the game directly, ignoring potential restrictions such as the age requirement.
- Since only registered players are expected to meet the `isOldEnough` condition, allowing unregistered users to play means that age restrictions are effectively unenforced.
- The system may rely on player data stored during registration for tracking purposes. If unregistered users are allowed to play, this could result in incorrect tracking of rewards, losses, or other metrics.

## Recommendation
The contract should verify that a user is registered before allowing them to deposit or request spins. Since the registration process already includes an age check, enforcing registration as a prerequisite ensures that only eligible players can participate. This can be done by requiring a mapping lookup that confirms the `msg.sender` is registered before proceeding with gameplay or deposits.

## Ludex Labs
Fixed in commit af74519.

## Cantina Managed
Fix ok.

---
