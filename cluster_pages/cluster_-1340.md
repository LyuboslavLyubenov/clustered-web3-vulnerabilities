# Cluster -1340

**Rank:** #270  
**Count:** 31  

## Label
Inadequate validation and state isolation in advanced order fulfillment lead to incorrect asset transfers, inconsistent recipient handling, and exploitable state mismatches during execution.

## Cluster Information
- **Total Findings:** 31

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

**Auto Label:** Failure to validate transfer outcomes or state conditions leads to inconsistent or incorrect state updates, enabling fund loss, reentrancy, or misrepresentation of asset balances.  

**Original Text Preview:**

## Summary

The function provided by the `RToken` contract to [rescue any ERC20 tokens](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L337-L341) that were sent to the contract by mistake is [only callable by](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L54-L57) [`LendingPool`](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L54-L57).
However, `LendingPool` [does not offer any function](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/pools/LendingPool/LendingPool.sol#L731-L734) to call the `RToken` contract, resulting in unrecoverable funds.

## Vulnerability Details

The `Rtoken` contract provides [a function to rescue any ERC20 tokens](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L337-L341) that have been sent to the contract directly by mistake.
This function is only callable by the configured `reservePool`, which is ensured by the [`onlyReservePool`](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L54-L57) [modifier](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L54-L57).
This will be [`LendingPool`](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/pools/LendingPool/LendingPool.sol#L33) in practice.

Looking at the `LendingPool` contract, it comes indeed with a [`rescueToken`](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/pools/LendingPool/LendingPool.sol#L731-L734) [function](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/pools/LendingPool/LendingPool.sol#L731-L734), however, that one only allows for rescuing funds from itself.
It does not provide the necessary functionality to recover funds from the `RToken` contract.

Here's what it looks like:

```Solidity
function rescueToken(address tokenAddress, address recipient, uint256 amount) external onlyOwner {
    require(tokenAddress != reserve.reserveRTokenAddress, "Cannot rescue RToken");
    IERC20(tokenAddress).safeTransfer(recipient, amount);
}
```

Notice how it uses `safeTransfer` to move funds from itself to the `recipient`, but it does not call into the `RToken` contract.

## Impact

ERC20 tokens, that are not the configured reserve asset, which are sent to the `RToken` contract by mistake are not recoverable until the owner of the protocol configures a new `reservePool` that provides the necessary functionality

## Tools Used

Manual review.

## Recommendations

There's two ways to go about this:

1. Either allow the `owner` of the protocol to call `rescueFunds` on `RToken` directly or
2. Extend `LendingPool`, which is the expected `reservePool` in production, to provide the necessary function.

I'd recommend going for option 1) simply because any change of `reservePool` could reintroduce this issue.
Here's the necessary change:

```diff
- function rescueToken(address tokenAddress, address recipient, uint256 amount) external onlyReservePool {
+ function rescueToken(address tokenAddress, address recipient, uint256 amount) external onlyOwner {
    if (recipient == address(0)) revert InvalidAddress();
    if (tokenAddress == _assetAddress) revert CannotRescueMainAsset();
    IERC20(tokenAddress).safeTransfer(recipient, amount);
}
```

## Relevant links

* [`RToken#rescueToken`](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L337-L341)
* [`RToken#onlyReservePool`](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L54-L57)
* [`RToken#setReservePool`](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L85-L90)
* [`LendingPool#rescueToken`](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/pools/LendingPool/LendingPool.sol#L731-L734)

* [Similar finding other contest](https://solodit.cyfrin.io/issues/m-4-rescuetokens-feature-is-broken-sherlock-notional-leveraged-vaults-pendle-pt-and-vault-incentives-git)

---
### Example 3

**Auto Label:** Misuse of state values and insufficient validation lead to incorrect order state representation, erroneous execution, or unintended fund transfers during order fulfillment.  

**Original Text Preview:**

## Context
- Eco7683OriginSettler.sol#L186-L190
- Eco7683OriginSettler.sol#L280-L282
- ERC7683.sol#L96-L101

## Summary
ERC-7683 intents can never be filled in the protocol because an incorrect contract address is used in `FillInstruction.destinationSettler`.

## Finding Description
Eco adds ERC-7683 compliant wrappers to allow ERC-7683 aware fillers to interoperate with the protocol. These wrappers are implemented in `Eco7683OriginSettler.sol` on the origin/source chains and `Eco7683DestinationSettler.sol` on the destination chains. `Eco7683OriginSettler` implements the following functions:
- `open()`
- `openFor()`
- `resolve()`
- `resolveFor()`

`Eco7683DestinationSettler` implements the `fill()` function per ERC-7683.

However, while resolving an `OnchainCrossChainOrder` to a `ResolvedCrossChainOrder` in `resolve()` and `resolveFor()`, Eco's inbox contract address is incorrectly used instead of `bytes20(uint160(Eco7683DestinationSettler))` for struct `FillInstruction.destinationSettler`, which is expected to be "The contract address that the order is meant to be filled on" for ERC-7683 intents.

## Impact Explanation
**Medium**, because fillers cannot use the ERC-7683 wrappers and instead have to rely on Eco's native intent logic.

## Likelihood Explanation
**High**, because an ERC-7683 aware filler picking up such an intent will attempt to call the corresponding ERC-7683 `fill()` on Eco's inbox contract, which will always revert because it does not implement the ERC-7683 `fill()` function.

## Recommendation
Consider using `bytes20(uint160(Eco7683DestinationSettler))` for struct `FillInstruction.destinationSettler` in `resolve()` and `resolveFor()` by appropriately making that address accessible in `Eco7683OriginSettler.sol`.

## Eco
To solve this, we did a bit of restructuring - the inbox is now the `Eco7683DestinationSettler`. The latter is now an abstract contract whose functionality is inherited by the inbox. See commit `2e7bc2f`.

## Cantina Managed
Reviewed that commit `2e7bc2f` fixes this issue with their above mentioned restructuring.

---
