# Cluster -1432

**Rank:** #403  
**Count:** 12  

## Label
Insufficient input validation leads to unauthorized operations, fund loss, or state manipulation through unchecked external calls or missing value verification.

## Cluster Information
- **Total Findings:** 12

## Examples

### Example 1

**Auto Label:** Insufficient input validation leads to state corruption, reward manipulation, or denial of service through unauthorized epoch or reward value manipulation.  

**Original Text Preview:**

##### Description

In the `Yield` contract, the `distributeYield` function is checking if the provided `payload.epoch` is greater than the last recorded epoch for the `receiver`. This allows epochs to be skipped.

Because of missing input validation, if the rewarder submits a payload with a significantly larger `epoch` value, future legitimate rewards will be unclaimable because the `epoch` has been artificially advanced.

  

`- contracts/core/Yield.sol`

```
    function distributeYield(bytes calldata data, bytes memory proofSignature) external notPaused onlyRewarder {
        bytes32 proofHash = keccak256(data);
        require(!trxns[proofHash], "!new txcn");
        validate(proofHash, proofSignature);

        RewardPayload memory payload = Codec.decodeReward(data);

        require(payload.epoch > epoch[payload.receiver], "!epoch");

        trxns[proofHash] = true;
        epoch[payload.receiver] = payload.epoch;
        
        if (payload.rewardType == 1) { // profit 
             _distributeYield(payload.receiver, payload.amount, true);
            profit[payload.receiver] += payload.amount;
        } else { // loss
            _distributeYield(payload.receiver, payload.amount, false);
            loss[payload.receiver] += payload.amount;
        }

        emit DistributeYield(IERC4626(payload.receiver).asset(), payload.receiver, payload.amount, payload.rewardType == 1);
    }
```

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:L/A:N/D:N/Y:N/R:N/S:C (3.1)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:L/A:N/D:N/Y:N/R:N/S:C)

##### Recommendation

It is recommended to add a more granular control, enforcing sequential epochs. This can be achieved by modifying the `require` statement as follows:

```
require(payload.epoch == epoch[payload.receiver] + 1, "!epoch");
```

##### Remediation

**SOLVED:** The **YieldFi team** has solved this issue as recommended. The commit hash for reference is `9e2e3842a08a08f0d1226300ef7eea3f8e68703d`.

##### Remediation Hash

<https://github.com/YieldFiLabs/contracts/commit/9e2e3842a08a08f0d1226300ef7eea3f8e68703d>

---
### Example 2

**Auto Label:** Insufficient input validation leads to state corruption, reward manipulation, or denial of service through unauthorized epoch or reward value manipulation.  

**Original Text Preview:**

## Vulnerability Report

**Severity:** Medium Risk  
**Context:** `core/blockchain.go#L1438-L1444`  
**Description:** Each checkpoint block, the chain goes through and deletes epochs based on the configured `EpochLimit`. Any value lower than 3 could delete required data.  
**Recommendation:** Enforce the minimum `EpochLimit` value of 3 on chain creation, while still allowing the value of 0 for unlimited `EpochLimit`.

---
### Example 3

**Auto Label:** Insufficient input validation leads to unauthorized operations, fund loss, or state manipulation through unchecked external calls or missing value verification.  

**Original Text Preview:**

##### Description

The return value of an external call is not stored in a local or state variable.

* **Code Location**

```
    function _supportMarket(MToken mToken) external returns (uint) {
        if (msg.sender != admin) {
            return fail(Error.UNAUTHORIZED, FailureInfo.SUPPORT_MARKET_OWNER_CHECK);
        }

        if (markets[address(mToken)].isListed) {
            return fail(Error.MARKET_ALREADY_LISTED, FailureInfo.SUPPORT_MARKET_EXISTS);
        }

        mToken.isMToken(); // Sanity check to make sure its really a MToken

        Market storage newMarket = markets[address(mToken)];
        newMarket.isListed = true;
        newMarket.collateralFactorMantissa = 0;

        _addMarketInternal(address(mToken));

        emit MarketListed(mToken);

        return uint(Error.NO_ERROR);
    }

```

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:F/S:C (0.0)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:F/S:C)

##### Recommendation

It is recommended to use a `require` statement due to the return value of an external call is not stored in a local or state variable.

  

### Remediation Plan

**SOLVED**: The `Moonwell Finance team` solved the issue by using `require` statement.

---
