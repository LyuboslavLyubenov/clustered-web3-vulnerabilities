# Cluster -1437

**Rank:** #67  
**Count:** 264  

## Label
Decimal precision mismatches in token handling lead to incorrect value calculations, flawed transfers, and financial losses due to improper scaling and lack of consistent decimal normalization across functions and chains.

## Cluster Information
- **Total Findings:** 264

## Examples

### Example 1

**Auto Label:** Failure to account for transfer fees or dynamic token balances leads to incorrect state updates, fund loss, or overstatements of user balances due to lack of on-chain validation and atomicity.  

**Original Text Preview:**

The `RivusTAO::wrap` function checks if the `wTAO` amount exceeds the `minStakingAmt` before fees are subtracted:

```solidity
File: RivusTAO.sol
887:   function wrap(uint256 wtaoAmount) public nonReentrant checkPaused returns (uint256) {
...
...
918:     // Ensure that at least 0.125 TAO is being bridged
919:     // based on the smart contract
920:     require(wtaoAmount > minStakingAmt, "Does not meet minimum staking amount");
921:
922:
923:     // Ensure that the wrap amount after free is more than 0
924:     (uint256 wrapAmountAfterFee, uint256 feeAmt) = calculateAmtAfterFee(wtaoAmount);
925:
926:     uint256 rsTAOAmount = getRsTAObyWTAO(wrapAmountAfterFee);
...
...
939:   }
```

This approach is flawed as it should ensure that the net amount after fees, which is actually being staked, meets the `minStakingAmt`. Performing the check prior to fee deductions can lead to scenarios where the actual staked amount is less than the intended minimum due to the fees subtracted afterward.

Adjust the validation process to check `minStakingAmt` after the fees have been calculated and deducted from the `wTAO` amount. This ensures that the amount effectively being staked still meets the minimum requirements stipulated by the protocol, preventing users from staking less than the minimum due to fees.

```diff
  function wrap(uint256 wtaoAmount) public nonReentrant checkPaused returns (uint256) {
    ...
    ...

--  // Ensure that at least 0.125 TAO is being bridged
--  // based on the smart contract
--  require(wtaoAmount > minStakingAmt, "Does not meet minimum staking amount");

    // Ensure that the wrap amount after free is more than 0
    (uint256 wrapAmountAfterFee, uint256 feeAmt) = calculateAmtAfterFee(wtaoAmount);

++  // Ensure that at least 0.125 TAO is being bridged
++  // based on the smart contract
++  require(wrapAmountAfterFee > minStakingAmt, "Does not meet minimum staking amount");

    uint256 rsTAOAmount = getRsTAObyWTAO(wrapAmountAfterFee);
    ...
    ...
  }
```

---
### Example 2

**Auto Label:** Failure to account for transfer fees or dynamic token balances leads to incorrect state updates, fund loss, or overstatements of user balances due to lack of on-chain validation and atomicity.  

**Original Text Preview:**

## Severity

Medium Risk

## Description

The `MultipliBridger` contract's deposit mechanism fails to properly account for fee-on-transfer (FoT) tokens. The vulnerability manifests in the `deposit` function, which invokes `TransferHelper.safeTransferFrom(token, msg.sender, address(this), amount)` to transfer tokens from the user to the contract. This implementation assumes the amount specified will exactly match the tokens received by the contract, which is incorrect for FoT tokens that deduct a transfer fee.

The contract then emits the `BridgedDeposit event` with the full pre-fee amount as the deposited value. Off-chain systems, including the sequencer and bridging back-end that monitor these events, will interpret this logged amount as the actual value received by the contract. This creates a discrepancy where users receive receipt tokens (`xTokens on L2`) based on the pre-fee amount while the contract holds less value than recorded.

## Location of Affected Code

File: [src/MultipliBridger.sol#L125](https://github.com/multipli-libs/multipli-bridger/blob/c5b4869fcf7349c9ce9b67ca26f6ae45cc7babce/src/MultipliBridger.sol#L125)

```solidity
function deposit(address token, uint256 amount) external {
  TransferHelper.safeTransferFrom(
    token,
    msg.sender,
    address(this),
    amount
  );
  emit BridgedDeposit(msg.sender, token, amount);
}
```

## Impact

When FoT tokens are deposited, the protocol will credit users with receipt tokens based on the pre-fee amount while holding less actual token value in the contract. This creates an accounting imbalance that could lead to liquidity shortfalls when processing withdrawals.

## Recommendation

The contract should implement FoT token handling by comparing the balance before and after transfers. For the `deposit` function, first record the contract's token balance, then perform the transfer, then verify the new balance to determine the actual received amount.

## Team Response

Fixed.

---
### Example 3

**Auto Label:** Failure to account for dynamic token balances during transfers or withdrawals, leading to incorrect state updates, fund loss, and exploitable inconsistencies in rebasing or fee-on-transfer token systems.  

**Original Text Preview:**

**Description:** As part of the implementation of the autonomous rebalance mechanism in `BunniHook::rebalanceOrderPreHook`, the hook balance of the order output token is set in transient storage before the order is executed. This occurs before the unlock callback, during which `hookHandleSwap()` is called within `_rebalancePrehookCallback()`:

```solidity
    function rebalanceOrderPreHook(RebalanceOrderHookArgs calldata hookArgs) external override nonReentrant {
        ...
        RebalanceOrderPreHookArgs calldata args = hookArgs.preHookArgs;

        // store the order output balance before the order execution in transient storage
        // this is used to compute the order output amount
        uint256 outputBalanceBefore = hookArgs.postHookArgs.currency.isAddressZero()
            ? weth.balanceOf(address(this))
            : hookArgs.postHookArgs.currency.balanceOfSelf();
        assembly ("memory-safe") {
@>          tstore(REBALANCE_OUTPUT_BALANCE_SLOT, outputBalanceBefore)
        }

        // pull input tokens from BunniHub to BunniHook
        // received in the form of PoolManager claim tokens
        // then unwrap claim tokens
        poolManager.unlock(
            abi.encode(
                HookUnlockCallbackType.REBALANCE_PREHOOK,
@>              abi.encode(args.currency, args.amount, hookArgs.key, hookArgs.key.currency1 == args.currency)
            )
        );

        // ensure we have at least args.amount tokens so that there is enough input for the order
@>      if (args.currency.balanceOfSelf() < args.amount) {
            revert BunniHook__PrehookPostConditionFailed();
        }
        ...
    }

    /// @dev Calls hub.hookHandleSwap to pull the rebalance swap input tokens from BunniHub.
    /// Then burns PoolManager claim tokens and takes the underlying tokens from PoolManager.
    /// Used while executing rebalance orders.
    function _rebalancePrehookCallback(bytes memory callbackData) internal {
        // decode data
        (Currency currency, uint256 amount, PoolKey memory key, bool zeroForOne) =
            abi.decode(callbackData, (Currency, uint256, PoolKey, bool));

        // pull claim tokens from BunniHub
@>      hub.hookHandleSwap({key: key, zeroForOne: zeroForOne, inputAmount: 0, outputAmount: amount});

        // lock BunniHub to prevent reentrancy
        hub.lockForRebalance(key);

        // burn and take
        poolManager.burn(address(this), currency.toId(), amount);
        poolManager.take(currency, address(this), amount);
    }
```

The transient storage is again queried in the call to `rebalanceOrderPostHook()` to ensure that only the specified amount of the output token (consideration item) is transferred from `BunniHook`:

```solidity
    function rebalanceOrderPostHook(RebalanceOrderHookArgs calldata hookArgs) external override nonReentrant {
        ...
        RebalanceOrderPostHookArgs calldata args = hookArgs.postHookArgs;

        // compute order output amount by computing the difference in the output token balance
        uint256 orderOutputAmount;
        uint256 outputBalanceBefore;
        assembly ("memory-safe") {
@>          outputBalanceBefore := tload(REBALANCE_OUTPUT_BALANCE_SLOT)
        }
        if (args.currency.isAddressZero()) {
            // unwrap WETH output to native ETH
            orderOutputAmount = weth.balanceOf(address(this));
            weth.withdraw(orderOutputAmount);
        } else {
            orderOutputAmount = args.currency.balanceOfSelf();
        }
@>      orderOutputAmount -= outputBalanceBefore;
        ...
```

However similar such validation is not applied to the input token (offer item). While the existing validation shown above is performed to ensure that the hook holds sufficient tokens to process the order, it fails to consider that the hook may hold funds designated to other recipients. Previously, per the Pashov Group finding H-04, autonomous rebalance was DoS’d by checking the token balance was strictly equal to the order amount but forgot to account for am-AMM fees and donations. The recommendation was to check that the contract’s token balance increase is equal to `args.amount`, not the total balance.

Given that am-AMM fees are stored as ERC-6909 balances within the `BunniHook`, these could be erroneously used as part of the rebalance order input amount. This could occur when the hook holds insufficient raw balance to cover the input amount, perhaps when rehypothecation is enabled and the `hookHandleSwap()` call pulls fewer tokens than expected due to the vault returning fewer tokens than specified. Instead, the input token balance should be set in transient storage before the unlock callback in the same way that `outputBalanceBefore` is. Then, the difference between the input token balances before the unlock callback and after should be validated to satisfy the order input amount.

**Impact:** am-AMM fees stored as ERC-6909 balance inside `BunniHook` could be erroneously used as rebalance input due to the incorrect balance check.

**Recommended Mitigation:** Consider modifying the validation performed after the unlock callback to:
```solidity
uint256 inputBalanceBefore = args.currency.balanceOfSelf();

// unlock callback

if (args.currency.balanceOfSelf() - inputBalanceBefore < args.amount) {
    revert BunniHook__PrehookPostConditionFailed();
}
```

**Bacon Labs:** Fixed in [PR \#98](https://github.com/timeless-fi/bunni-v2/pull/98) and [PR \#133](https://github.com/timeless-fi/bunni-v2/pull/133).

**Cyfrin:** Verified, stricter input amount validation has been added to `BunniHook::rebalanceOrderPreHook`.

---
