# Cluster -1252

**Rank:** #206  
**Count:** 54  

## Label
Inconsistent or missing validation of zero or invalid deposit amounts across asset types leads to unauthorized fund transfers, state corruption, or failed status updates, enabling front-running, misallocation, or permanent fund trapping.

## Cluster Information
- **Total Findings:** 54

## Examples

### Example 1

**Auto Label:** **Unchecked return values and input validation failures lead to unauthorized state modifications, fund locks, and incorrect debt calculations.**  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/964 

## Found by 
0xEkko, 0xc0ffEE, 0xgee, Kvar, SarveshLimaye, Z3R0, benjamin\_0923, dmdg321, ggg\_ttt\_hhh, h2134, jokr, mussucal, newspacexyz, oxelmiguel, patitonar, rudhra1749, sil3th, wickie, xiaoming90

### Summary

The function `CoreRouter::repayBorrowInternal()` only updates user's `borrowBalance` (which is the borrow balance same-chain). This can cause cross chain repayment to update wrong borrow balance

### Root Cause

The function [`_repayCrossChainBorrowInternal()`](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L368) is used within the flow of function [`repayCrossChainBorrow()`](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L161) and [`_handleLiquidationSuccess()`](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L464). The function `_repayCrossChainBorrowInternal()` calls internal function `_handleRepayment()`, which will external call to `CoreRouter::repayCrossChainLiquidation()`. The function `CoreRouter::repayCrossChainLiquidation()` calls internal function `repayBorrowInternal()`. Here exists problem that the cross chain borrow information is not updated, but the same-chain borrow balance.
Indeed, on the dest chain, the borrow information `crossChainCollateral` should be updated.
```solidity
    function repayBorrowInternal(
        address borrower,
        address liquidator,
        uint256 _amount,
        address _lToken,
        bool _isSameChain
    ) internal {
        address _token = lendStorage.lTokenToUnderlying(_lToken);

        LTokenInterface(_lToken).accrueInterest();

        uint256 borrowedAmount;

        if (_isSameChain) {
            borrowedAmount = lendStorage.borrowWithInterestSame(borrower, _lToken);
        } else {
            borrowedAmount = lendStorage.borrowWithInterest(borrower, _lToken);
        }

        require(borrowedAmount > 0, "Borrowed amount is 0");

        uint256 repayAmountFinal = _amount == type(uint256).max ? borrowedAmount : _amount;

        // Transfer tokens from the liquidator to the contract
        IERC20(_token).safeTransferFrom(liquidator, address(this), repayAmountFinal);

        _approveToken(_token, _lToken, repayAmountFinal);

        lendStorage.distributeBorrowerLend(_lToken, borrower);

        // Repay borrowed tokens
        require(LErc20Interface(_lToken).repayBorrow(repayAmountFinal) == 0, "Repay failed");

        // Update same-chain borrow balances
        if (repayAmountFinal == borrowedAmount) {
@>            lendStorage.removeBorrowBalance(borrower, _lToken);
@>            lendStorage.removeUserBorrowedAsset(borrower, _lToken);
        } else {
@>            lendStorage.updateBorrowBalance(
                borrower, _lToken, borrowedAmount - repayAmountFinal, LTokenInterface(_lToken).borrowIndex()
            );
        }

        // Emit RepaySuccess event
        emit RepaySuccess(borrower, _lToken, repayAmountFinal);
    }
```

### Internal Pre-conditions

NA

### External Pre-conditions

NA

### Attack Path

NA

### Impact

- Potential unable to repay cross chain borrow and liquidate cross chain borrow
- If the function `repayBorrowInternal()` successfully executed, then the same chain borrow balance is also updated, which is incorrectly accounting

### PoC

_No response_

### Mitigation

Since the function `_updateRepaymentState()` is executed later to update cross chain borrow information, then the function `repayBorrowInternal()` should consider not update borrow balance when `_isSameChain == false`

---
### Example 2

**Auto Label:** **Unchecked deposit validation leading to state inconsistency, overflow, or unauthorized exposure.**  

**Original Text Preview:**

The `depositIntoAaveV3()` function allows addresses with the `CONFIGURATOR_ROLE` to deposit arbitrary amounts of `wstETH` into the Aave V3 pool. However, it does not check whether the resulting position will exceed the `_maxPositionThreshold + BPS_BUFFER`, which is the upper bound that triggers an emergency scale-down in the `scaleDown()` function.

This omission allows the `CONFIGURATOR_ROLE` to unintentionally deposit an amount that will immediately make the position non-compliant with the threshold limit. As a result, an automated scale-down might be triggered soon after deposit, leading to unnecessary gas usage and increased operational overhead.

```solidity
    function depositIntoAaveV3(
        uint256 amount
    ) external onlyRole(CONFIGURATOR_ROLE) {
        require(amount > 0, InvalidZeroAmount());

        IERC20(WST_ETH).forceApprove(_aaveV3Pool, amount);
        IPool(_aaveV3Pool).supply(WST_ETH, amount, address(this), 0);

        emit DepositIntoAaveV3(amount);
    }
```

Recommendation:
Add a check in `depositIntoAaveV3()` to ensure that the resulting position percentage remains below `_maxPositionThreshold + BPS_BUFFER`.

---
### Example 3

**Auto Label:** **Unchecked deposit validation leading to state inconsistency, overflow, or unauthorized exposure.**  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-04-zetachain-cross-chain-judging/issues/45 

## Found by 
BrothersInCode, g

### Summary

### Description

Any user can initiate a deposit on the SUI network by calling the `gateway.move::deposit` function:

```javascript
public entry fun deposit<T>(
    gateway: &mut Gateway,
    coin: Coin<T>,
    receiver: String,
    ctx: &mut TxContext,
) {
    let amount = coin.value();
    let coin_name = coin_name<T>();

    check_receiver_and_deposit_to_vault(gateway, coin, receiver);

    // Emit deposit event
    event::emit(DepositEvent {
        coin_type: coin_name,
        amount: amount,
        sender: tx_context::sender(ctx),
        receiver: receiver,
    });
}
```

This function in turn calls`check_receiver_and_deposit_to_vault`:

```javascript
// Validates receiver and deposits the coin into the vault
fun check_receiver_and_deposit_to_vault<T>(gateway: &mut Gateway, coin: Coin<T>, receiver: String) {
    assert!(receiver.length() == ReceiverAddressLength, EInvalidReceiverAddress);
    assert!(is_whitelisted<T>(gateway), ENotWhitelisted);
    assert!(!gateway.deposit_paused, EDepositPaused);

    // Perform the deposit
    let coin_name = coin_name<T>();
    let vault = bag::borrow_mut<String, Vault<T>>(&mut gateway.vaults, coin_name);
    balance::join(&mut vault.balance, coin.into_balance());
}
```

This helper function performs a series of checks, including:

- Verifying the receiver address format
- Checking that the coin type is whitelisted
- Ensuring that deposits are not currently paused

However, SUI's deposit flow does not perform a minimum deposit amount/fee check.

Note that in all of the other implementations (EVM, Solana, TON) these checks are correctly performed.  


### Root Cause

The root cause of this issue is that there is no minimum amount/fee check enforced inside the following [function](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/e3caacef006bcef90af3035ed79d95075e31264d/protocol-contracts-sui/sources/gateway.move#L181-L199)

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

1. Any user can call `gateway.move::deposit` with `coin` amount set to 0 or a neglible amount
2. User repeats this as many times as he wants



### Impact

There are several impacts here:

- Without a minimum amount or fee requirement, users can submit empty or negligible-value deposits, which will overwhelm the node’s processing logic which in turn will drain the nodes’ funds and cause a potential DOS.
- SUI's nodes have a maximum event processing limit of 50. A malicious user could exploit this by spamming deposit transactions, quickly hitting the limit with their own events and easily causing DOS for other users

### PoC
- Paste the following test-function inside `gateway_tests.move`:
```javascript
#[test]
fun test_deposit_zero_amount() {
    let mut scenario = ts::begin(@0xA);
    setup(&mut scenario);

    ts::next_tx(&mut scenario, @0xA);
    {
        let mut gateway = scenario.take_shared<Gateway>();
        let admin_cap = ts::take_from_address<AdminCap>(&scenario, @0xA);

        let coin = coin::mint_for_testing<SUI>(0, scenario.ctx()); // deposit 0 amount
        let eth_addr = ValidEthAddr.to_string().to_ascii();

        deposit(&mut gateway, coin, eth_addr, scenario.ctx());

        ts::return_to_address(@0xA, admin_cap);
        ts::return_shared(gateway);
    };
    ts::end(scenario);
}
```
- run `sui move test test_deposit_zero_amount`
- Logs:
```diff
Running Move unit tests
[ PASS    ] gateway::gateway_tests::test_deposit_zero_amount
Test result: OK. Total tests: 1; passed: 1; failed: 0
```

### Mitigation

Ensure that users are not able to perform empty deposit calls.

---
