# Cluster -1327

**Rank:** #12  
**Count:** 821  

## Label
Insufficient state validation and incorrect power calculations allow users to manipulate voting power and locked supply, bypassing caps and distorting governance incentives through biased, time-ignoring, or double-counted updates.

## Cluster Information
- **Total Findings:** 821

## Examples

### Example 1

**Auto Label:** Failure to verify ownership or state changes enables attackers to duplicate votes, manipulate quorums, or inflate voting power, undermining governance integrity through inconsistent or unverified state transitions.  

**Original Text Preview:**

<https://github.com/code-423n4/2025-05-blackhole/blob/92fff849d3b266e609e6d63478c4164d9f608e91/contracts/BlackGovernor.sol# L52>

<https://github.com/code-423n4/2025-05-blackhole/blob/92fff849d3b266e609e6d63478c4164d9f608e91/contracts/BlackGovernor.sol# L91>

### Finding description

* **Severity**: Medium
* **Affected contracts**:

  + `contracts/BlackGovernor.sol`
  + `contracts/VotingEscrow.sol` (as the source of `getPastTotalSupply` and `smNFTBalance`)
  + `contracts/MinterUpgradeable.sol` (as the target of the `nudge()` function)

The mechanism for proposals in `BlackGovernor.sol` has a fundamental mismatch between how the `proposalThreshold` is determined and how the `quorum` is calculated.

1. **Proposal Threshold**: Determined by `proposalThreshold()`, which requires a percentage (default 0.2%, configurable by `team` up to 10%) of the *total current `veBLACK` voting power*. This includes voting power from both regular `veNFT` locks and Supermassive NFTs (smNFTs).

   
```

   // contracts/BlackGovernor.sol
   function proposalThreshold() public view override returns (uint256) {
      return (token.getPastTotalSupply(block.timestamp) * proposalNumerator) / PROPOSAL_DENOMINATOR;
   }
   // token.getPastTotalSupply() resolves to VotingEscrow.totalSupplyAtT(timestamp)
   
```

2. **Quorum**: Determined by `quorum()`, which requires 4% of `token.getsmNFTPastTotalSupply()`. This `getsmNFTPastTotalSupply()` resolves to `VotingEscrow.smNFTBalance`.

   `VotingEscrow.smNFTBalance` represents the cumulative sum of *principal `BLACK` tokens ever burned* to create or upgrade to smNFTs. This `smNFTBalance` value only ever increases. The 4% of this principal amount is then used as the target for the *total voting power* that must be cast in favor of a proposal.

   
```

   // contracts/BlackGovernor.sol
   function quorum(uint256 blockTimestamp) public view override returns (uint256) {
       return (token.getsmNFTPastTotalSupply() * quorumNumerator()) / quorumDenominator(); // quorumNumerator is 4
   }
   // token.getsmNFTPastTotalSupply() resolves to VotingEscrow.smNFTBalance
   
```

   This creates a situation where the ability to propose is based on overall `veBLACK` voting power distribution, but the ability for a proposal to pass (quorum) is heavily tied to a growing, historical sum of burned `BLACK` for smNFTs, which might not correlate with active smNFT voting participation or the total active voting power.

### Risk and Impact

The primary risk is a **Denial of Service (DoS)** for the `MinterUpgradeable.nudge()` function, which is the *only* function `BlackGovernor.sol` can call.

* **Governance deadlock scenario (Medium risk)**: As the protocol matures, `VotingEscrow.smNFTBalance` (total `BLACK` principal burned for smNFTs) can grow significantly. If this balance becomes very large, the 4% quorum target (in terms of required voting power) can become unachievably high for the following reasons:

  + Active smNFT holders might be a small fraction of the total `smNFTBalance` contributors (e.g., due to inactive early minters).
  + The collective voting power of active smNFT holders who support a given proposal might be insufficient to meet this high quorum.
  + A user or group could meet the `proposalThreshold()` using voting power from regular `veBLACK` locks, but their proposal would consistently fail if the smNFT-derived quorum is not met by other voters. This leads to the `nudge()` function becoming unusable, preventing any future adjustments to the tail emission rate via this governor.
* **Governance spam**: A secondary consequence is the potential for governance “spam,” where proposals are repeatedly created meeting the threshold but are destined to fail due to the quorum structure, causing on-chain noise.
* **Contrast - low quorum scenario (Informational)**: Conversely, if `smNFTBalance` is very low (e.g., early in the protocol, or if smNFTs are unpopular), the quorum can be trivially met, potentially allowing a small group (that meets proposal threshold) to easily control the `nudge()` function. This aspect was previously noted but provides context to the design’s sensitivity to `smNFTBalance`.
* **Illustrative deadlock scenario**:

  1. The `smNFTBalance` in `VotingEscrow.sol` has grown to a large number (e.g., 10,000,000 `BLACK` principal burned).
  2. The quorum target, in terms of voting power, becomes 4% of this, i.e., equivalent to the voting power derived from 400,000 `BLACK` (if it were all smNFTs with average lock/bonus).
  3. A group of users (“Proposers”) accumulates 0.2% of the total `veBLACK` voting power (e.g., from a mix of regular and some smNFT locks) and creates a proposal to `nudge()` emissions.
  4. Many of the smNFTs contributing to the `smNFTBalance` were created by users who are now inactive or do not vote on this proposal.
  5. The active smNFT holders, plus the Proposers’ own smNFT voting power, sum up to less than the 400,000 `BLACK` equivalent voting power required for quorum.
  6. The proposal fails. This cycle can repeat, rendering the `nudge()` function effectively disabled.

### Recommended mitigation steps

The current quorum mechanism presents a significant risk to the intended functionality of `BlackGovernor.sol`. Consider the following:

* **Align quorum base**: Modify the quorum calculation to be based on a metric more aligned with active participation or total voting power, similar to the `proposalThreshold`. For example, base the quorum on a percentage of `token.getPastTotalSupply(block.timestamp)` (total `veBLACK` voting power).
* **Adaptive quorum**: Implement an adaptive quorum mechanism. This could involve:

  + A quorum that adjusts based on recent voting participation in `BlackGovernor` proposals.
  + A system where the contribution of an smNFT to the `smNFTBalance` (for quorum calculation purposes, not its actual voting power) decays over time if the smNFT is inactive in governance.
* **Dual quorum condition**: Consider requiring the lesser of two quorum conditions to be met: e.g., (4% of `smNFTBalance`-derived voting power) OR (X% of total `veBLACK` voting power). This provides a fallback if `smNFTBalance` becomes disproportionately large or inactive.
* **Documentation and monitoring**: If the current design is retained, thoroughly document the rationale and potential long-term implications. Continuously monitor `smNFTBalance` growth versus active smNFT voting participation and total `veBLACK` supply to assess if the `nudge()` function’s viability is threatened.

**Blackhole marked as informative**

---

---
### Example 2

**Auto Label:** Failure to verify ownership or state changes enables attackers to duplicate votes, manipulate quorums, or inflate voting power, undermining governance integrity through inconsistent or unverified state transitions.  

**Original Text Preview:**

<https://github.com/code-423n4/2025-05-blackhole/blob/main/contracts/VotingEscrow.sol# L1263-L1279>

<https://github.com/code-423n4/2025-05-blackhole/blob/main/contracts/libraries/VotingBalanceLogic.sol# L20-L43>

The `getsmNFTPastVotes` function checks what was the voting power of the some account at specific point in time.
```

function  getsmNFTPastVotes(

address account,

uint timestamp

) public  view  returns (uint) {

uint32 _checkIndex = VotingDelegationLib.getPastVotesIndex(

cpData,

account,

timestamp

);

// Sum votes

uint[] storage _tokenIds = cpData

.checkpoints[account][_checkIndex].tokenIds;

uint votes = 0;

for (uint i = 0; i < _tokenIds.length; i++) {

uint tId = _tokenIds[i];

if (!locked[tId].isSMNFT) continue;

// Use the provided input timestamp here to get the right decay

votes =

votes +

VotingBalanceLogic.balanceOfNFT(

tId,

timestamp,

votingBalanceLogicData

);

}

return votes;
```

We can see in the snippet above that if specific token is not smNFT **RIGHT NOW** (as locked mapping stores current state of `tokenId`); it skips its voting power in the calculation. However, the function **DOES NOT** take into the consideration that specific `tokenId` can be smNFT right now, but could be permanent/not permanent NFT at `timestamp - timestamp`. This means that if some NFTs at `timestamp X` was not-permanent locked position, now it’s smNFT (it is possible inside the VotingEscrow to update NFTs to smNFTs), the voting power from the `timestamp X` will be added to the `votes` variable calculation which is incorrect as at this point in time X, the nft WAS NOT smNFT.

The biggest impact here is that `BlackGovernor` contract uses the `getsmNFTPastVotes` to determine the voting power of the account at some point in time (and the calculation is incorrect). This leads to some users having calculated bigger voting power than they should have leading to for example proposals being proposed from users who SHOULD NOT have that ability or votes casted using bigger power than actual.

### Recommended mitigation steps

When calculating check whether at `timestamp` NFT was actually smNFT.

**[Blackhole mitigated](https://github.com/code-423n4/2025-06-blackhole-mitigation?tab=readme-ov-file# mitigation-of-high--medium-severity-issues)**

**Status:** Mitigation confirmed. Full details in reports from [maxvzuvex](https://code4rena.com/audits/2025-06-blackhole-mitigation-review/submissions/S-64) and [rayss](https://code4rena.com/audits/2025-06-blackhole-mitigation-review/submissions/S-55).

---

---
### Example 3

**Auto Label:** Failure to verify ownership or state changes enables attackers to duplicate votes, manipulate quorums, or inflate voting power, undermining governance integrity through inconsistent or unverified state transitions.  

**Original Text Preview:**

<https://github.com/code-423n4/2025-05-blackhole/blob/main/contracts/governance/Governor.sol# L184-L191>

<https://github.com/code-423n4/2025-05-blackhole/blob/main/contracts/governance/Governor.sol# L680-L684>

<https://github.com/code-423n4/2025-05-blackhole/blob/main/contracts/MinterUpgradeable.sol# L138-L155>

### Finding description and impact

The `BlackGovernor` should be able to, based on user votes determine the status of the proposal and then execute it. We can see from the code that if proposal succeeds, then the emissions inside the `MinterUpgradeable` should decrease by 1%, if proposal is defeated it should decrease by 1%, otherwise, if quorum is not reached or the `forVotes`/`againstVotes` are not big enough compared to abstain votes (which means proposal state is expired) the emissions should stay the same.

However, when determining the status of the proposal the following check is made:
```

// quorum reached is basically a percentage check which is of the number specified in constructor of the L2GovernorVotesQuorumFraction(4) // 4%

if (_quorumReached(proposalId) && _voteSucceeded(proposalId)) {

return ProposalState.Succeeded;

} else  if (_quorumReached(proposalId) && _voteDefeated(proposalId)) {

return ProposalState.Defeated;

} else {

return ProposalState.Expired;

}
```

From the code we can see that if quorum is not reached it, the proposal is considered Expired after the voting period has ended. However, due to the fact that `_quorumReachedFunction` is using ONLY the `forVotes` and `abstainVotes`, which is incorrect based on the functionality that protocol wants to introduce, we can imagine the following scenario:

* The emissions are high, and almost all of the users agree that they should decrease the emissions rate.
* 99% of votes are `againstVotes` as almost everybody agrees that emissions should decrease.
* The voting period for the proposal ends

  + User tries to execute the proposal expecting that the emissions rate will decrease as 99% of the votes were against and this amount of votes should meet the quorum.
  + The function executes but it calculates the state as Expired because according to it quorum has not been reached as against votes DO NOT count into the quorum.
  + The emissions rate stay the same even though “all” of the voting users agreed that it should decrease.

Quorum reached snippet:
```

function  _quorumReached(

uint256 proposalId

) internal  view virtual override returns (bool) {

ProposalVote  storage proposalvote = _proposalVotes[proposalId];

return

quorum(proposalSnapshot(proposalId)) <=
//@audit NO AGAINST VOTES
proposalvote.forVotes + proposalvote.abstainVotes;

}
```

We can see that evaluation whether quorum was reached or not is incorrect based on the functionality that protocol wants to achieve leading to, for example, decrease of the emissions rate by the `BlackGovernor` being not possible, when almost everybody agrees to do so. Based on the provided facts, I think this should be Medium severity finding as it disrupts the emissions of the BlackToken.

### Recommended mitigation steps

When evaluating whether quorum was reached, take against votes into the account.

**[Blackhole mitigated](https://github.com/code-423n4/2025-06-blackhole-mitigation?tab=readme-ov-file# mitigation-of-high--medium-severity-issues)**

**Status:** Mitigation confirmed. Full details in reports from [rayss](https://code4rena.com/audits/2025-06-blackhole-mitigation-review/submissions/S-41) and [lonelybones](https://code4rena.com/audits/2025-06-blackhole-mitigation-review/submissions/S-8).

The sponsor team requested that the following note be included:

> This issue originates from the upstream codebase, inherited from ThenaV2 fork. Given that ThenaV2 has successfully operated at scale for several months without incident, we assess the severity of this issue as low. The implementation has been effectively battle-tested in a production environment, which significantly reduces the practical risk associated with this finding.
> Reference: <https://github.com/ThenafiBNB/THENA-Contracts/blob/main/contracts/governance/Governor.sol# L668>

---

---
