# Cluster -1057

**Rank:** #439  
**Count:** 8  

## Label
Vulnerabilities allow actors to bypass or avoid fee payments through mechanism flaws in fee handling, front-running, or off-chain dependency reliance, leading to potential fund loss or protocol misuse.

## Cluster Information
- **Total Findings:** 8

## Examples

### Example 1

**Auto Label:** Vulnerabilities stem from inadequate fee enforcement, allowing malicious actors to manipulate or evade fees, leading to unfair user losses, revenue misalignment, and compromised economic fairness.  

**Original Text Preview:**

## Severity

Medium Risk

## Description

When using the protocol, the user will call `GlueXRouter.swap()` and set the `RouteDescription` as the parameter. The user can control the routing fee and can set it to 1 wei to evade fees.

```solidity
struct RouteDescription {
    IERC20 inputToken;
    IERC20 outputToken;
    address payable inputReceiver;
    address payable outputReceiver;
    uint256 inputAmount;
    uint256 outputAmount;
@>  uint256 routingFee;
    uint256 effectiveOutputAmount;
    uint256 minOutputAmount;
    bool isPermit2;
}
```

The `routingFee` cannot be 0, but it can be 1 wei.

```solidity
function swap(
    Executor executor,
@>  RouteDescription calldata desc,
    Interaction[] calldata interactions
) external payable returns (uint256 finalOutputAmount) {
    // code

    require(desc.routingFee > 0, "Negative routing fee");

    // code
}
```

## Location of Affected Code

File: [contracts/GluexRouter.sol#L35](https://github.com/gluexprotocol/gluex_router/blob/1df4eaef9c3063a3171961e1f8bba3eb83c6b7e1/contracts/GluexRouter.sol#L35)

File: [contracts/GluexRouter.sol#L90](https://github.com/gluexprotocol/gluex_router/blob/1df4eaef9c3063a3171961e1f8bba3eb83c6b7e1/contracts/GluexRouter.sol#L90)

## Impact

Users can evade fees.

## Recommendation

Consider setting the fee at a base percent `(maybe 0.1 - 1%)` and every time a `swap()` happens, a percentage of the fees will go directly to the fee treasury.

## Team Response

Acknowledged, the risk will be mitigated off-chain, with access granted only to a select group of users.

---
### Example 2

**Auto Label:** Vulnerabilities stem from inadequate fee enforcement, allowing malicious actors to manipulate or evade fees, leading to unfair user losses, revenue misalignment, and compromised economic fairness.  

**Original Text Preview:**

## Exploitation of Protocol Fee

The current protocol design enables the admin to exploit `set_protocol_fee` by invoking it to front-run liquidity provider deposits or withdrawals to manipulate the protocol fees. This may result in a scenario in which the admin adjusts the protocol fees to an extremely high value (e.g., 100%).

## Code Snippet

```rust
s-controller/src/processor/set_protocol_fee.rs

pub fn process_set_protocol_fee(
    accounts: &[AccountInfo],
    args: SetProtocolFeeIxArgs,
) -> ProgramResult {
    let (
        accounts,
        SetProtocolFeeIxArgs {
            new_trading_protocol_fee_bps,
            new_lp_protocol_fee_bps,
        },
    ) = verify_set_protocol_fee(accounts, args)?;
    [...]
    Ok(())
}
```

Consequently, users depositing into the liquidity pool may receive fewer liquidity pool tokens than expected, impacting their share of the pool and potential rewards. Users withdrawing from the pool may receive fewer underlying assets than expected due to the higher fee, reducing the value of their holdings.

## Remediation

As a preventive measure, introduce a new parameter (example: `min_lp_token_out`), ensuring that users receive a minimum amount of liquidity pool tokens by calculating the tokens to distribute based on the new protocol fees and ensuring that the output satisfies the specified minimum.

## Patch

Fixed in `4f71ef6`.

---
### Example 3

**Auto Label:** Vulnerabilities allow actors to bypass or avoid fee payments through mechanism flaws in fee handling, front-running, or off-chain dependency reliance, leading to potential fund loss or protocol misuse.  

**Original Text Preview:**

## EAS.sol

## Description
EAS supports delegated attestations/revocations via signatures submitted to relayers. However, the trust assumptions among attesters/revokers, relayers, and resolvers are outside the scope of protocol implementation.

Relayers can become unresponsive, either accidentally or intentionally, to prevent subsequent delegated attestations/revocations from succeeding because of the sequential nonce enforcement in signatures. They may also front-run each other to revert other multi-bundles (e.g., `multiAttestByDelegation()` and `multiRevokeByDelegation()` functions can be front-run with split transactions), causing griefing. This may require attesters, revokers, and relayers to resubmit signatures with the appropriate nonces.

Relayers are expected to make any required payments to resolvers based on offchain agreements with attesters, revokers, and resolvers. They receive any unused payments as refunds. This allows for potential collusion between relayers and resolvers at the cost of attesters and revokers, depending on the resolver logic. Relayers may also cause a loss or lock of funds for attesters, revokers, or resolvers by not honoring their offchain agreements.

## Impact
- **Potential loss of attester/revoker funds** in the worst case.
- **Griefing of attesters, resolvers, and peers** by relayers dropping or delaying transactions.

**Likelihood:** Undetermined  
**Impact:** Medium  
**Severity:** Low

## Recommendation
Document the risks to users for awareness and consider mitigative designs (e.g., capturing the payment amount in signatures) to reduce the risk from cheating or griefing relayers.

## EAS
Fixed in PR#13.

## Cantina Managed
Reviewed that PR#13 captures the relayer/resolver payment amount in attest/revoke signatures.

---
