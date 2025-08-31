# Cluster -1215

**Rank:** #335  
**Count:** 20  

## Label
Cross-chain trust vulnerabilities arising from insufficient validation of chain context, state mutations, and message integrity, enabling spoofing, fund redirection, and denial of service.

## Cluster Information
- **Total Findings:** 20

## Examples

### Example 1

**Auto Label:** Insufficient validation of external inputs leads to gas wastage, balance draining, revenue loss, and unauthorized fund transfers through unchecked off-chain or public data.  

**Original Text Preview:**

## Severity

Informational

## Description

In the requestWithdraw process, users submit a withdrawal request by providing a proof
and public signals which get sent in a Telegram group. Relayers monitor this group and use the pro-
vided information to call the on-chain withdraw function.
However, without proper off-chain validation, bad actors can flood the Telegram group with invalid
or duplicate requests, causing relayers to waste gas attempting failed transactions.

## Recommendation

To prevent this, off-chain validation should include:
Nullifier Existence – Ensuring the provided nullifier corresponds to a valid deposit.
Correct Recipient – Verifying that the recipient address matches the expected one for the nullifier.
Withdrawal Status – Checking if the nullifier has already been used for a withdrawal.

## Team Response

Fixed.

---
### Example 2

**Auto Label:** Insufficient validation of external inputs leads to gas wastage, balance draining, revenue loss, and unauthorized fund transfers through unchecked off-chain or public data.  

**Original Text Preview:**

## Data Validation Issue

**Difficulty:** Undetermined  
**Type:** Data Validation  
**Target:** contracts/pixelswap_streampool.tact  

## Description
To create a new token, a user must send a `CreateToken` message to the Stream Pool contract with enough value to pay for the token creation fee. This fee is also used as a mitigation feature to disincentivize the creation of excessively many tokens, given that the tokens are stored in a mapping structure inside the Stream Pool contract’s storage.

In the `CreateToken` message handler, the message value is checked to ensure that it is enough to pay the creation fee, but it doesn’t take into account the gas costs needed to execute the transaction. This allows any user to create new tokens without paying for the computation gas required to create them, using the Stream Pool contract balance to pay for it.

[Redacted]

## Exploit Scenario
Alice decides to drain the balance of a `PixelswapStreamPool` contract. To do this, she starts sending `CreateToken` messages to the Stream Pool with a value just above (i.e., by 1 nanoton) the token creation fee. Repeating this will drain the `PixelswapStreamPool` contract balance, which will be used to pay for all the `CreateToken` message processing. Admins will need to send TON to it to continue paying the storage fee.

## Recommendations
- **Short term:** Modify the value check in the `CreateToken` message receiver to ensure the user provides enough TON for the required gas.
- **Long term:** Expand checks in the test cases to ensure that the TON balance of the system smart contracts does not go down after user actions that do not transfer TON.

## Fix Review Status
After conducting a fix review, the team determined that this issue has been resolved.

---
### Example 3

**Auto Label:** Cross-chain trust vulnerabilities arising from insufficient validation of chain context, state mutations, and message integrity, enabling spoofing, fund redirection, and denial of service.  

**Original Text Preview:**

##### Description

The `CCIPAdapter` contract in the LucidLabs protocol uses Chainlink's Cross-Chain Interoperability Protocol (CCIP) to facilitate cross-chain communication. However, the implementation fails to handle exceptions gracefully in the receiving contracts, specifically in `VotingControllerUpgradeable` and `AssetController`.

```
function _ccipReceive(Client.Any2EVMMessage memory any2EvmMessage) internal override {
    _registerMessage(bytes32ToAddress(_originSender), _callData, chainId);
}
```

  

This function calls `registerMessage()` on `VotingControllerUpgradeable` and `AssetController`, which can revert due to various reasons:

```
function castCrossChainVote(...) external {
    //E @AUDIT can revert because of state(proposalId) , timepoint is not the good
    if ((adapter != msg.sender) || (state(proposalId) != ProposalState.Active) || (proposalSnapshot(proposalId) != timepoint) || (chainTokens[chainId] != sourceToken))
        revert Governor_WrongParams();
// ...
    _countVote(proposalId, voter, support, votes, voteData, chainId);
// ...
}

function _countVote(
        uint256 proposalId,
        address account,
        uint8 support,
        uint256 totalWeight,
        bytes memory voteData, //E when called from LucidGovernor{Timelock} it is not implemented => _countVoteNominal is called
        uint256 chainId 
    ) internal virtual {
        
        if (totalWeight == 0) revert GovernorCrossCountingFractionalUpgradeable_NoWeight();

        if (_proposalVotersWeightCast[_getVoterKey(proposalId, account, chainId)] >= totalWeight) { revert("GovernorCountingFractional: all weight cast");}

        // ... // 
    }
```

```
function receiveMessage(...) public override nonReentrant {
// ...
    uint256 _currentLimit = mintingCurrentLimitOf(msg.sender);
    if (_currentLimit < transfer.amount) revert IXERC20_NotHighEnoughLimits();
// ...
}
```

  

These unhandled exceptions can cause CCIP messages to fail, potentially triggering Chainlink's manual execution mode after the 8-hour Smart Execution window. This will block subsequent messages, leading to a Denial of Service in cross-chain communication until the failed message can be executed.

  

The unhandled exceptions in CCIP message processing result in a vulnerability that disrupts cross-chain operations. This issue leads to:

  

1. Failure in updating cross-chain voting data, compromising the integrity of the governance system.
2. Blockage of asset transfers between chains, affecting the core functionality of the protocol.
3. Potential permanent DoS of the CCIP pathway **until the blocked message can pass**

##### BVSS

[AO:A/AC:L/AX:M/C:N/I:N/A:H/D:N/Y:N/R:N/S:C (6.3)](/bvss?q=AO:A/AC:L/AX:M/C:N/I:N/A:H/D:N/Y:N/R:N/S:C)

##### Recommendation

It is recommended to implement proper exception handling in the receiving contracts or to handle messages received/execution in 2 ways.

##### Remediation

**RISK ACCEPTED:** As the probability of this risk happening is low and even if the impact is high, the **Lucid Labs team** will leave the code as is, and will use the other adapters in case of this happening.

##### References

[LucidLabsFi/demos-contracts-v1/contracts/modules/chain-abstraction/adapters/ccip/CCIPAdapter.sol#L109](https://github.com/LucidLabsFi/demos-contracts-v1/blob/main/contracts/modules/chain-abstraction/adapters/ccip/CCIPAdapter.sol#L109)

---
