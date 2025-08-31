# Cluster -1041

**Rank:** #4  
**Count:** 1068  

## Label
Failure to validate or remove token state entries enables malicious actors to manipulate contract state, leading to denial of service, incorrect rewards, or unauthorized transfers through inadequate input checks or missing state updates.

## Cluster Information
- **Total Findings:** 1068

## Examples

### Example 1

**Auto Label:** Failure to properly validate or handle token transfer outcomes, leading to unauthorized transfers, reentrancy, or system lockouts due to missing return value checks or inadequate error handling.  

**Original Text Preview:**

## Severity

Medium Risk

## Description

The contract performs an ERC20 token `transferFrom()` without verifying its success. While most major tokens revert on failure, some non-ERC20 standard tokens technically allow them to return false instead of reverting.

This introduces a risk where Non-Compliant Tokens may fail silently, while the protocol assumes success, potentially leading to missing or undelivered funds.

## Location of Affected Code

File: [src/CredifiERC20Adaptor.sol#L620](https://github.com/credifi/contracts-audit/blob/ba976bad4afaf2dc068ca9dcd78b38052d3686e3/src/CredifiERC20Adaptor.sol#L620)

```solidity
function _performLoanRepayment( address user, IndividualLoan storage loan, uint256 repayAmount, address payer, bool requireFullRepayment ) internal returns (uint256 actualRepayAmount, uint256 remainingDebt, bool fullyRepaid) {
   // code

   // Transfer repay tokens from payer to this contract
   borrowToken.transferFrom(payer, address(this), actualRepayAmount);
   // code
}
```

## Recommendation

Consider using `safeTransferFrom()` instead of `transferFrom()`:

```solidity
// Use OpenZeppelin's SafeERC20
using SafeERC20 for IERC20;

// Replace transfer with:
borrowToken.safeTransferFrom(payer, address(this), actualRepayAmount);
```

## Team Response

Fixed.

---
### Example 2

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
### Example 3

**Auto Label:** Failure to properly validate or handle token transfer outcomes, leading to unauthorized transfers, reentrancy, or system lockouts due to missing return value checks or inadequate error handling.  

**Original Text Preview:**

Currently, the `returnExcessiveBond` function is available inside the `WorkerRegistration`. In the event that the bond is ever reduced from the system, the excess amount can be claimed and returned to the creator. However, if the bond ever increases, there is no direct way for workers to update the bond, it has to go through the whole deregister and register process, which requires waiting for a lock period. This will add a time delay for the actual bond inside the system to match the TVL calculation. Consider to add function to update bond in case of the bond ever increased, similar to `returnExcessiveBond`.

---
