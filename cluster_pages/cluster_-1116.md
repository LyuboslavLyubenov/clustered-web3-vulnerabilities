# Cluster -1116

**Rank:** #138  
**Count:** 103  

## Label
Missing boundary checks and invalid state validations enable unauthorized minting beyond per-generation limits, leading to infinite generations, inflated prices, and inconsistent token counting.

## Cluster Information
- **Total Findings:** 103

## Examples

### Example 1

**Auto Label:** Missing state updates and control flow flaws lead to unbounded token emissions, unauthorized minting, and failed emission responses, enabling infinite supply or privilege escalation.  

**Original Text Preview:**

<https://github.com/code-423n4/2025-05-blackhole/blob/main/contracts/governance/Governor.sol# L308-L330>

### Finding description and impact

`MinterUpgradeable` has `nudge` function that is called by the `BlackGovernor` contract. The nudge function then check the `status` of the `BlackGovernor` and based on status, updates the `tailEmissionRate` of the contract; however, from analyzing the `BlackGovernor` contract (which means also analyzing contracts inside the `Governor.sol` ), we can see that the `status` variable **DOES NOT CHANGE EVER**. This leads to the fact that even though proposal has succeeded meaning the `tailEmissionRate` should be equal to `PROPOSAL_INCREASE`, it goes to else branch and `tailEmissionRate` never changes.

This is because the status variable inside `BlackGovernor/Governor` is SHADOWED in the `execute` function, which can be easily checked using even Solidity Visual Developer extension.
```

function  nudge() external {

address _epochGovernor = _gaugeManager.getBlackGovernor();

require(msg.sender == _epochGovernor, "NA");

//@audit STATUS NEVER GETS UPDATED !!!

IBlackGovernor.ProposalState _state = IBlackGovernor(_epochGovernor)

.status();

require(weekly < TAIL_START);

uint256 _period = active_period;

require(!proposals[_period]);

if (_state == IBlackGovernor.ProposalState.Succeeded) {

tailEmissionRate = PROPOSAL_INCREASE;

} else  if (_state == IBlackGovernor.ProposalState.Defeated) {

tailEmissionRate = PROPOSAL_DECREASE;

} else {

tailEmissionRate = 10000;

}

proposals[_period] = true;

}
```


```

function  execute(

address[] memory targets,

uint256[] memory values,

bytes[] memory calldatas,

bytes32 epochTimeHash

) public  payable virtual override returns (uint256) {

uint256 proposalId = hashProposal(

targets,

values,

calldatas,

epochTimeHash

);

//@audit variable is shadowed

ProposalState status = state(proposalId);
```

This vulnerability should be High severity, as the core functionality of the protocol is not working. The status is never updated leading to `tailEmissionRate` always being the same.

### Recommended mitigation steps

Get rid of the shadowing and update the status variable.

### Proof of Concept

This POC is written in foundry, in order to run it, one has to create foundry project, download dependencies, create remappings and change imports in some of the file to look inside the “src” folder.

The POC is the simple version of the finding, in order to run it one has to first change the code of the Governor, so that other vulnerability in the code is fixed, as without that change proposals DO NOT WORK. Look at the [`getVotes` inside the `BlackGovernor` incorrectly provides `block.number` instead of `block.timestamp`, leading to complete DOS of proposal functionality](https://code4rena.com/audits/2025-05-blackhole/submissions/F-163) finding.

Also to see what’s happening one might add `console.logs` to minter upgradeable and Governor contract that is used inside `BlackGovernor`. We can see that even though the state of the proposal should be `X`, the status variable is always default value.
```

pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";
import {Black} from "../src/Black.sol";
import {MinterUpgradeable} from "../src/MinterUpgradeable.sol";
import {RewardsDistributor} from "../src/RewardsDistributor.sol";
import {PermissionsRegistry} from "../src/PermissionsRegistry.sol";
import {TokenHandler} from "../src/TokenHandler.sol";
import {VoterV3} from "../src/VoterV3.sol";
import {VotingEscrow} from "../src/VotingEscrow.sol";
import {AutoVotingEscrowManager} from "../src/AVM/AutoVotingEscrowManager.sol";
import {GaugeManager} from "../src/GaugeManager.sol";
import {PairGenerator} from "../src/PairGenerator.sol";
import {GaugeFactory} from "../src/factories/GaugeFactory.sol";
import {PairFactory} from "../src/factories/PairFactory.sol";
import {GaugeFactoryCL} from "../src/AlgebraCLVe33/GaugeFactoryCL.sol";
import {BlackGovernor} from "../src/BlackGovernor.sol";
import {IBlackHoleVotes} from "../src/interfaces/IBlackHoleVotes.sol";
import {IMinter} from "../src/interfaces/IMinter.sol";
import {BlackTimeLibrary} from "../src/libraries/BlackTimeLibrary.sol";

contract GovernanceStatusNotUpdatingTest is Test {
    Black public black;
    RewardsDistributor public rewardsDistributor;
    MinterUpgradeable public minterUpgradeable;
    PermissionsRegistry public permissionsRegistry;
    TokenHandler public tokenHandler;
    AutoVotingEscrowManager public avm;
    VotingEscrow public votingEscrow;
    VoterV3 public voter;
    GaugeManager public gaugeManager;
    GaugeFactory public gaugeFactory;
    PairGenerator public pairGenerator;
    PairFactory public pairFactory_1;
    PairFactory public pairFactory_2CL;
    GaugeFactoryCL public gaugeFactoryCL;
    BlackGovernor public blackGovernor;
    address admin = address(0xAD);
    address protocol = address(0x1);
    address userA = address(0xA);
    address userB = address(0xB);
    function setUp() public {
        vm.startPrank(admin);

        black = new Black();
        permissionsRegistry = new PermissionsRegistry();
        string memory GARole = string(bytes("GAUGE_ADMIN"));
        permissionsRegistry.setRoleFor(admin, GARole);
        tokenHandler = new TokenHandler(address(permissionsRegistry));
        voter = new VoterV3();//needs to be initialized
        avm = new AutoVotingEscrowManager();//needs to be initialized
        votingEscrow = new VotingEscrow(address(black), address(0), address(avm));

        rewardsDistributor = new RewardsDistributor(address(votingEscrow));

        gaugeFactoryCL = new GaugeFactoryCL();
        gaugeFactoryCL.initialize(address(permissionsRegistry));

        gaugeFactory = new GaugeFactory();
        pairGenerator = new PairGenerator();
        pairFactory_1 = new PairFactory();
        pairFactory_1.initialize(address(pairGenerator));
        pairFactory_2CL = new PairFactory();
        pairFactory_2CL.initialize(address(pairGenerator));

gaugeManager = new GaugeManager();
        gaugeManager.initialize(address(votingEscrow), address(tokenHandler),address( gaugeFactory),address( gaugeFactoryCL), address(pairFactory_1), address(pairFactory_2CL), address(permissionsRegistry));

        avm.initialize(address(votingEscrow),address(voter), address(rewardsDistributor));

        minterUpgradeable = new MinterUpgradeable();
        minterUpgradeable.initialize(address(gaugeManager), address(votingEscrow), address(rewardsDistributor));

        blackGovernor = new BlackGovernor(IBlackHoleVotes(votingEscrow), address(minterUpgradeable));
        gaugeManager.setBlackGovernor(address(blackGovernor));

        voter = new VoterV3();
        voter.initialize(address(votingEscrow), address(tokenHandler), address(gaugeManager), address(permissionsRegistry));

rewardsDistributor.setDepositor(address(minterUpgradeable));
        gaugeManager.setVoter(address(voter));
        gaugeManager.setMinter(address(minterUpgradeable));
        black.mint(admin, 10_000_000e18);
        black.setMinter(address(minterUpgradeable));

vm.stopPrank();
    }

    function test_statusNotUpdatingVote() public {
        vm.startPrank(admin);
        address[] memory claimants;
        uint[] memory amounts;

        minterUpgradeable._initialize(claimants, amounts, 0);//zero % ownership of top protcols
        uint userABal = 1_000_000e18;
        black.transfer(userA, userABal);

        vm.stopPrank();
        vm.startPrank(userA);
        uint WEEK = 1800;

for(uint i = 0; i< 67; i++) {
           skip(WEEK);
            minterUpgradeable.update_period();
             uint currentEpoch = minterUpgradeable.epochCount();
            uint weekly = minterUpgradeable.weekly();
        }

         black.approve(address(votingEscrow), type(uint).max);
        votingEscrow.create_lock(userABal, 4 * 365 days, true);

        skip(3600);
        address[] memory targets = new address[](1);
        uint256[] memory values = new uint256[](1);
        bytes[] memory calldatas = new bytes[](1);
        targets[0] = address(minterUpgradeable);
        values[0] = 0;
        calldatas[0] = abi.encodeWithSelector(IMinter.nudge.selector);
        bytes32 epochTimehash = bytes32(BlackTimeLibrary.epochNext(block.timestamp));
        uint proposalId = blackGovernor.propose(
            targets,
            values,
            calldatas,
            "Nudge proposal"
        );

        skip(blackGovernor.votingDelay()+1);

        blackGovernor.castVote(proposalId, 1);

        (
            uint256 againstVotes,
            uint256 forVotes,
            uint256 abstainVotes
        )= blackGovernor.proposalVotes(proposalId);

        console.log("Against Votes: ", againstVotes);
        console.log("For Votes: ", forVotes);
        console.log("Abstain Votes: ", abstainVotes);
        skip(blackGovernor.votingPeriod() + 1);

        blackGovernor.execute(targets, values, calldatas, epochTimehash);
        //Look at the logs and status of the Governor

    }
}
```

**[Blackhole mitigated](https://github.com/code-423n4/2025-06-blackhole-mitigation?tab=readme-ov-file# mitigation-of-high--medium-severity-issues)**

**Status:** Mitigation confirmed. Full details in reports from [rayss](https://code4rena.com/audits/2025-06-blackhole-mitigation-review/submissions/S-42), [lonelybones](https://code4rena.com/audits/2025-06-blackhole-mitigation-review/submissions/S-10) and [maxvzuvex](https://code4rena.com/audits/2025-06-blackhole-mitigation-review/submissions/S-33).

---

---
### Example 2

**Auto Label:** Missing state updates and control flow flaws lead to unbounded token emissions, unauthorized minting, and failed emission responses, enabling infinite supply or privilege escalation.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** High

## Description

`Minter.updatePeriod()` is meant to mint Kitten tokens for the `RebaseReward` and `Voter` contracts once per period. However, this function misses updating the `lastMintedPeriod` variable. As a result, once the first period has passed, the following condition will always be true:

```solidity
        if (currentPeriod > lastMintedPeriod) {
```

This will allow minting Kitten tokens indefinitely, leading to an unlimited supply of Kitten tokens.

### Proof of concept

Add the following code to the file `TestVoter.t.sol` and run `forge test -f https://rpc.hyperliquid.xyz/evm --mc TestVoter --mt test_audit_unlimitedMinting`.

```solidity
    function test_audit_unlimitedMinting() public {
        test_Vote();
        vm.warp(block.timestamp + 1 weeks);

        uint256 totalSupplyBefore = kitten.totalSupply();
        for (uint256 i; i < 10; i++) {
            minter.updatePeriod();
            uint256 totalSupplyAfter = kitten.totalSupply();
            assertGt(totalSupplyAfter, totalSupplyBefore);
            totalSupplyBefore = totalSupplyAfter;
        }
    }
```

## Recommendations

```diff
            emit Mint(
                msg.sender,
                emissions,
                circulatingSupply(),
                _tailEmissions
            );
+
+           lastMintedPeriod = currentPeriod;

            return true;
```

---
### Example 3

**Auto Label:** **Improper bounds checking leads to unbounded or under-distributed token issuance, enabling supply overflow, over-issuance, and loss of control.**  

**Original Text Preview:**

`maxMint` currently returns `type(uint256).max`, which is incorrect. When users deposit to the vault, it checks if the `assets` deposited do not exceed `maxPerDeposit`. Instead of returning `type(uint256).max`, `maxMint` should return the shares corresponding to `maxPerDeposit`.

```diff
    function maxMint(address) public view virtual override returns (uint256) {
-        return type(uint256).max;
+        _convertToShares(maxPerDeposit, Math.Rounding.Down);
    }
```

---
