# Cluster -1282

**Rank:** #319  
**Count:** 22  

## Label
Insufficient input validation leads to out-of-bounds access, invalid data storage, and incorrect state enforcement, compromising data integrity and enabling denial-of-service or inconsistent behavior.

## Cluster Information
- **Total Findings:** 22

## Examples

### Example 1

**Auto Label:** Failure to validate input size enables memory exhaustion and data injection, leading to denial-of-service and arbitrary payload exploitation through unbounded resource allocation.  

**Original Text Preview:**

##### Description

The SSP relay API endpoint have insufficient data sanitization and lack of proper input size validation. Specifically, the parameters can accept excessively large input without any restrictions. This absence of data validation poses a potential risk, as it may lead to potential denial-of-service (DoS) attacks or resource exhaustion by sending large payloads, impacting the availability and performance of the service.

Attackers can exploit this weakness by crafting large payloads to consume excessive server resources, degrade the user experience, or even crash the service. Additionally, the lack of input sanitization might expose the application to further risks, such as injection-based attacks.

##### Proof of Concept

![](https://halbornmainframe.com/proxy/audits/images/67879bfc6c7eae81f01091cd)![](https://halbornmainframe.com/proxy/audits/images/67879c136c7eae81f0109362)

##### Score

Impact: 3  
Likelihood: 3

##### Recommendation

It is recommended to implement robust input validation and sanitization for all incoming data at the server level. Specifically:

• Enforce strict size limits on all fields to prevent excessively large payloads from being processed.

• Validate the data type and format of all parameters to ensure they meet expected criteria.

• Sanitize input data to mitigate potential injection attacks and ensure safe handling of user-provided data.

• Return appropriate error messages for inputs that exceed the defined limits, and log such attempts for further investigation.

• Consider implementing rate limiting to prevent abuse through high-frequency requests with large payloads.

##### Remediation

**SOLVED:** The **InFlux Technologies team** addressed the issue by implementing limit and checks on the parameters.

##### Remediation Hash

a17bd2596fb89c3a851b6ff8dab334055895031f

---
### Example 2

**Auto Label:** Unbounded resource consumption via malformed or excessive inputs, leading to denial-of-service through improper validation and lack of limits on input size or frame volume.  

**Original Text Preview:**

##### Description

A vulnerability was discovered with the implementation of the HTTP/2 protocol in Golang prior to 1.21.9 and 1.22.2 versions. The current version used in app chain is 1.21.

An attacker can cause the HTTP/2 endpoint to read arbitrary amounts of header data by sending an excessive number of CONTINUATION frames. This causes excessive CPU consumption of the receiver device since there is no sufficient limitation on the amount of frames. Thus, It could be exploited to cause DoS.

Please note that, many HTTP/2 implementations (including Golang) did not properly limit the amount of CONTINUATION frames within a single stream.

**References:**

<https://kb.cert.org/vuls/id/421644>

<https://www.cve.org/CVERecord?id=CVE-2023-45288>

##### Score

Impact:   
Likelihood:

##### Recommendation

The issue is fixed in both Golang 1.21.9 and 1.22.2. However, If you are intending to use 1.21.X, It is recommended upgrading to 1.21.11 (the latest of 1.21.X) since it has some other security/bug fixes in net/http package.

##### Remediation

**ACKNOWLEDGED:** The **Artela team** acknowledged the issue.

---
### Example 3

**Auto Label:** Unbounded array growth and insufficient access controls enable denial-of-service attacks and unauthorized state modifications through excessive resource consumption or privilege escalation.  

**Original Text Preview:**

##### Description

The functions `pullFundsFromSingleStrategy`, `pushFundsIntoSingleStrategy`, and `addStrategy` fail to implement checks for out-of-bounds access on the array of strategies with which the vault can interact.

```
/**
     * @notice Pulls funds back from a single strategy into the vault.
     * @dev Can only be called by the vault owner.
     * @param index_ The index of the strategy from which to pull funds.
     */
    function pullFundsFromSingleStrategy(uint256 index_) external onlyOwner {
        Strategy memory strategy = strategies[index_];
        // slither-disable-next-line unused-return
        strategy.strategy.redeem(strategy.strategy.balanceOf(address(this)), address(this), address(this));
        if (!IERC20(asset()).approve(address(strategy.strategy), 0)) revert ERC20ApproveFail();
    }

    /**
     * @notice Pushes funds from the vault into a single strategy based on its allocation.
     * @dev Can only be called by the vault owner. Reverts if the vault is idle.
     * @param index_ The index of the strategy into which to push funds.
     */
    function pushFundsIntoSingleStrategy(uint256 index_) external onlyOwner {
        if (vaultIdle) revert VaultIsIdle();
        Strategy memory strategy = strategies[index_];
        // slither-disable-next-line unused-return
        strategies[index_].strategy.deposit(
            totalAssets().mulDiv(strategy.allocation.amount, 10_000, Math.Rounding.Floor), address(this)
        );
    }
```

When the `replace_` parameter is set to true, the function did not perform a check on the index.

```
function addStrategy(uint256 index_, bool replace_, Strategy calldata newStrategy_)
        external
        nonReentrant
        onlyOwner
        takeFees
    {
        //Ensure that allotments do not total > 100%
        uint256 allotmentTotals = 0;
        uint256 len = strategies.length;
        for (uint256 i; i < len; i++) {
            allotmentTotals += strategies[i].allocation.amount;
        }

        if (replace_) {
            if (allotmentTotals - strategies[index_].allocation.amount + newStrategy_.allocation.amount > 10000) {
                revert AllotmentTotalTooHigh();
            }
            // slither-disable-next-line unused-return
            strategies[index_].strategy.redeem(
                strategies[index_].strategy.balanceOf(address(this)), address(this), address(this)
            );
            if (!IERC20(asset()).approve(address(strategies[index_].strategy), 0)) revert ERC20ApproveFail();

            emit StrategyReplaced(strategies[index_], newStrategy_);

            strategies[index_] = newStrategy_;
            if (!IERC20(asset()).approve(address(newStrategy_.strategy), type(uint256).max)) revert ERC20ApproveFail();
        } else {
            if (allotmentTotals + newStrategy_.allocation.amount > 10000) {
                revert AllotmentTotalTooHigh();
            }
            strategies.push(newStrategy_);
            if (!IERC20(asset()).approve(address(newStrategy_.strategy), type(uint256).max)) revert ERC20ApproveFail();
        }
        emit StrategyAdded(newStrategy_);
    }
```

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:N/S:U (0.0)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:N/S:U)

##### Recommendation

It is advisable to implement out-of-bounds checks when accessing arrays in Solidity. Additionally, providing an error message when attempting to fetch data beyond the array's limit can inform the user about the cause of the execution reversion.

##### Remediation

**SOLVED :** The **Concrete team** solved the issue by adding array index checks.

##### Remediation Hash

<https://github.com/Blueprint-Finance/money-printer/pull/26/commits/167bbc342f1b836763767b0dfbc503bc4240f7ad>

---
