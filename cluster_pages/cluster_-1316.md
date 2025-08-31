# Cluster -1316

**Rank:** #432  
**Count:** 9  

## Label
Front-running attacks exploit transaction ordering and inadequate fee controls, allowing malicious actors to profit from or disrupt legitimate claims by manipulating gas pricing and reward mechanisms.

## Cluster Information
- **Total Findings:** 9

## Examples

### Example 1

**Auto Label:** Front-running attacks exploit transaction ordering and inadequate fee controls, allowing malicious actors to profit from or disrupt legitimate claims by manipulating gas pricing and reward mechanisms.  

**Original Text Preview:**

## Context
(No context files were provided by the reviewer)

## Description
Any entity operating a mech can create simple requests and set the mech they are operating as the priority mech, allowing it to deliver right away and thus increase its `mapMechKarma`. On the other hand, `mapRequesterMechKarma` for a specific requester would be harder to manipulate since if the mech does not deliver quality data, the requester would stop creating requests that would have that specific mech as the priority mech (linked by the service id). Although this can also be manipulated if one considers the frontrunning issues related to `deliverMarketplaceWithSignatures`.

## Recommendation
A monitoring and analysis should be set up to shine a light on the above behaviors. It would also be best to document the above.

## Valory
There are several ways to game karma, and we acknowledge that we need to include this into our documentation.

## Cantina Managed
Acknowledged by Valory team.

---
### Example 2

**Auto Label:** Gas price manipulation enables MEV exploitation and front-running, allowing attackers to trigger slashing or seize bonds by distorting gas price signals and manipulating transaction ordering.  

**Original Text Preview:**

##### Description

Operators in Holograph perform tasks using the `executeJob()` function with bridged bytes from the source chain. If the primary operator fails to execute the job within the allocated block, a bond is taken and transferred to the operator doing the job. The presence of a gas spike is checked with `tx.gasprice`:

```
require(gasPrice >= tx.gasprice, "HOLOGRAPH: gas spike detected");
```

However, attackers can exploit this by submitting a flashbots bundle with `executeJob()` at a low gas price and an additional bribe to miners, enabling them to steal the bond from honest operators. This is not theoretical, as MEV bots exploit such opportunities regularly:

* See [coinbase.transfer()](https://docs.flashbots.net/flashbots-auction/advanced/coinbase-payment).
* See [Bundle selection](https://docs.flashbots.net/flashbots-auction/advanced/bundle-pricing#bundle-ordering-formula).

Therefore, a dishonest operator can seize an honest operator's bond even when gas prices are high.

##### Score

Impact:   
Likelihood:

##### Recommendation

Avoid using current `tx.gasprice` for previous block gas price inference. It is recommended to use a gas price oracle instead.

##### Remediation

**PENDING:** The **Holograph team** will consider addressing this in a future release.

---
### Example 3

**Auto Label:** Gas price manipulation enables MEV exploitation and front-running, allowing attackers to trigger slashing or seize bonds by distorting gas price signals and manipulating transaction ordering.  

**Original Text Preview:**

##### Description

In the `HolographOperator`contract, gas price spikes expose the selected operator to frontrunning and slashing vulnerabilities. The current code includes:

```
require(gasPrice >= tx.gasprice, "HOLOGRAPH: gas spike detected");
// operator slashing logic
_bondedAmounts[job.operator] -= amount;
_bondedAmounts[msg.sender] += amount;
```

This mechanism aims to prevent operators from being slashed due to gas spikes. However, it allows other operators to front-run by submitting their transactions with a higher gas price, resulting in the selected operator being slashed if they delay their transaction.

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:N/A:N/D:H/Y:N/R:N/S:U (7.5)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:N/A:N/D:H/Y:N/R:N/S:U)

##### Recommendation

Adjust the operator node software to queue transactions immediately with the gas price specified in `bridgeInRequestPayload` during a gas spike.

##### Remediation

**RISK ACCEPTED:** The **Holograph team** accepted the risk of this finding.

---
