# Cluster -1360

**Rank:** #149  
**Count:** 94  

## Label
Inadequate input validation and arithmetic safeguards lead to incorrect state transitions, financial misalignment, and potential overflow or underflow, enabling malicious actors to manipulate fees, liquidity, or ratios.

## Cluster Information
- **Total Findings:** 94

## Examples

### Example 1

**Auto Label:** Unbounded array growth leading to gas exhaustion and denial-of-service, enabling fund locking and unauthorized state manipulation.  

**Original Text Preview:**

The `maxUnstakeRequests` variable helps limit the number of `unstakeRequests` a user can have. This variable is validated in the `requestUnstake()` function in lines `RivusTAO#L607-609`:

```solidity
File: RivusTAO.sol
555:   function requestUnstake(uint256 rsTAOAmt) public payable nonReentrant checkPaused {
...
...
606:     if (!added) {
607:       require(
608:         unstakeRequests[msg.sender].length < maxUnstakeRequests,
609:         "Maximum unstake requests exceeded"
610:       );
611:       unstakeRequests[msg.sender].push(
612:         UnstakeRequest({
613:           amount: rsTAOAmt,
614:           taoAmt: outWTaoAmt,
615:           isReadyForUnstake: false,
616:           timestamp: block.timestamp,
617:           wrappedToken: wrappedToken
618:         })
619:       );
...
629:     }
...
...
638:   }
```

The problem is that `maxUnstakeRequests` can be modified via the `setMaxUnstakeRequest()` function, causing this variable validation to not work as expected. Consider the following scenario:

1. The owner calls `setMaxUnstakeRequest()` with a value of `2`.
2. `Account1` deposits `10e9 wTAO tokens`, obtaining `10e9 rsTAO tokens`.
3. For some reason, `Account1` performs an `unstaking` of `1e9 rsTAO` and another of `2e9 rsTAO`, obtaining `3e9 wTAO - fees`.
4. The owner modifies `setMaxUnstakeRequest()` and decreases it to a value of `1`.
5. `Account1` performs another `unstaking` of `3e9 rsTAO` and `4e9 rsTAO` in separate transactions, resulting in `Account1` having 2 `requestStaking`. **This is incorrect** as `Account1` should be limited to only 1 `requestUnstaking` since `maxUnstakeRequests` was decreased in `step 4`.

The following test demonstrates the previous scenario:

```js
it("maxUnstakeRequests is not used when user have already unstakeRequests", async function () {
  // setup
  const { owner, account1, account2, account3, wTAO, rsTAO, wrsTAO } =
    await loadFixture(deployContracts);
  await wTAO.setBridge(owner.address);
  const froms = ["from0", "from1", "from1", "from1"];
  const tos = [
    owner.address,
    account1.address,
    account2.address,
    account3.address,
  ];
  const amounts = [
    ethers.parseUnits("100", 9),
    ethers.parseUnits("10", 9),
    ethers.parseUnits("10", 9),
    ethers.parseUnits("10", 9),
  ];
  await wTAO.bridgedTo(froms, tos, amounts);
  console.log(
    "\nAccount1 wTAO balance: ",
    ethers.formatUnits(await wTAO.balanceOf(account1.address), 9)
  );
  console.log(
    "Account1 rsTAO balance: ",
    ethers.formatUnits(await rsTAO.balanceOf(account1.address), 9)
  );
  //
  // 1. Owner set max unstakeRequest to 2 just for the testing purposes
  await rsTAO.setMaxUnstakeRequest(2);
  //
  // 2. Account1 stakes wTAO
  let amountToStake = ethers.parseUnits("10", 9);
  console.log("\nStaking", ethers.formatUnits(amountToStake, 9), "wTAO...");
  await wTAO.connect(account1).approve(rsTAO.target, amountToStake);
  await rsTAO.connect(account1).wrap(amountToStake);
  console.log(
    "Account1 wTAO balance: ",
    ethers.formatUnits(await wTAO.balanceOf(account1.address), 9)
  );
  console.log(
    "Account1 rsTAO balance: ",
    ethers.formatUnits(await rsTAO.balanceOf(account1.address), 9)
  );
  //
  // 3. Account1 request unstake 1 wTAO
  console.log("\nRequest unstake 1 wTAO...");
  await rsTAO.connect(account1).requestUnstake(ethers.parseUnits("1", 9), {
    value: ethers.parseEther("0.003"),
  });
  //
  // 4. Account1 request unstake 2 wTAO
  console.log("\nRequest unstake 2 wTAO...");
  await rsTAO.connect(account1).requestUnstake(ethers.parseUnits("2", 9), {
    value: ethers.parseEther("0.003"),
  });
  //
  // 5. Account1 unstake both `requestUnstakes`
  console.log("\nOwner approves all the `Account1` requestUnstake");
  const userRequests = [
    { user: account1.address, requestIndex: 0 },
    { user: account1.address, requestIndex: 1 },
  ];
  await wTAO.approve(rsTAO.target, ethers.parseUnits("10", 9));
  await rsTAO.approveMultipleUnstakes(userRequests);
  console.log("\nAccount1 unstake both requests");
  await rsTAO.connect(account1).unstake(0);
  await rsTAO.connect(account1).unstake(1);
  console.log(
    "Account1 wTAO balance: ",
    ethers.formatUnits(await wTAO.balanceOf(account1.address), 9)
  );
  console.log(
    "Account1 rsTAO balance: ",
    ethers.formatUnits(await rsTAO.balanceOf(account1.address), 9)
  );
  //
  // 6. Owner set max unstakeRequest to 1 but the account1 can still use 2 request unstakes
  await rsTAO.setMaxUnstakeRequest(1);
  console.log("\nRequest unstake 3 wTAO...");
  await rsTAO.connect(account1).requestUnstake(ethers.parseUnits("1", 9), {
    value: ethers.parseEther("0.003"),
  });
  console.log("\nRequest unstake 4 wTAO...");
  await rsTAO.connect(account1).requestUnstake(ethers.parseUnits("2", 9), {
    value: ethers.parseEther("0.003"),
  });
  let getUserRequests = await rsTAO.getUnstakeRequestByUser(account1.address);
  console.log(getUserRequests.length);
});
```

It is recommended to ensure that `maxUnstakeRequests` is not exceeded within the section where empty `unstakeRequests` are reused in lines `RivusTAO#L577-L602`.

```solidity
File: RivusTAO.sol
555:   function requestUnstake(uint256 rsTAOAmt) public payable nonReentrant checkPaused {
...
...
576:     // Loop throught the list of existing unstake requests
577:     for (uint256 i = 0; i < length; i++) {
578:       uint256 currAmt = unstakeRequests[msg.sender][i].amount;
579:       if (currAmt > 0) {
580:         continue;
581:       } else {
582:         // If the curr amt is zero, it means
583:         // we can add the unstake request in this index
584:         unstakeRequests[msg.sender][i] = UnstakeRequest({
585:           amount: rsTAOAmt,
586:           taoAmt: outWTaoAmt,
587:           isReadyForUnstake: false,
588:           timestamp: block.timestamp,
589:           wrappedToken: wrappedToken
590:         });
591:         added = true;
592:         emit UserUnstakeRequested(
593:           msg.sender,
594:           i,
595:           block.timestamp,
596:           rsTAOAmt,
597:           outWTaoAmt,
598:           wrappedToken
599:         );
600:         break;
601:       }
602:     }
...
...
```

---
### Example 2

**Auto Label:** Timing-dependent logic flaws leading to invalid state transitions, incorrect calculations, and exploitable pricing or reward mechanisms.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Medium

## Description

The option premium is calculated using the input `period`, while the actual duration until the maturity date is rounded up to the end of the day which is typically longer than the given `period`. This allows traders to get longer option durations than what they pay for.

The premium calculation in `Strategy._calculatePremium()` uses the input `period` in seconds to calculate the option price:

```solidity
function _calculatePremium(uint256 amount, uint256 period, uint256 preciseCurrentPrice, uint256 strike, uint256 iv)
    internal
    view
    returns (uint256 premium)
{
    (uint256 callPremium, uint256 putPremium) =
@>      StrategyStorage.layout().setUp.base.bs.optionPrices(period, iv, preciseCurrentPrice, strike, 0);
```

However, the maturity date is rounded up to the end of the day in `PositionParams._computeMaturityDate()`:

```solidity
function _computeMaturityDate(uint256 period) internal view returns (uint32) {
    uint256 maturityTimestamp = block.timestamp + period;
    maturityTimestamp -= (maturityTimestamp % 1 days);
    maturityTimestamp += 1 days - 1;
    return uint32(maturityTimestamp);
}
```

Therefore, the gap between the input `period` and the actual duration until maturity date can be up to `86399` seconds, which can be a significant difference for short-period positions.

Consider the following scenario:

1. Current time: `1743638400` (Thu Apr 03 2025 00:00:00).
2. User inputs period = 7 days = `604800` seconds.
3. Actual maturity: `1744329599` (Thu Apr 10 2025 23:59:59).

Input period: `604800` seconds
Actual period: `1744329599 - 1743638400` = `691199` seconds (7 days + 86399 seconds ≈ 8 days).

The premium is calculated using the input period (`604800` seconds) in the Black-Scholes formula, resulting in a lower premium as it underestimates the time until maturity.

Below is an example output for a 7-day `CALL` option (3500 USD spot price, 10% relative strike, 48.9% IV).
This results in approximately 23% underpricing for this `CALL` option position.

```bash
Logs:
  Period: 604800 sec
  Maturity: 1744329599
  Actual Period: 691199 sec
  --- Premium: period ---
  call: 8963759233891000000
  put: 358963759233891000000
  --- Premium: maturity - uint32(block.timestamp)---
  call: 11675104193168500000
  put: 361675104193168500000

  Premium difference: 2711344959277500000
```

## Recommendation

Update the premium calculation to use the actual duration until the maturity date.

---
### Example 3

**Auto Label:** Inadequate input validation and arithmetic safeguards lead to incorrect state transitions, financial misalignment, and potential overflow or underflow, enabling malicious actors to manipulate fees, liquidity, or ratios.  

**Original Text Preview:**

## Diﬃculty: Low

## Type: Data Validation

## Description
The pool contract’s reserve ratio validation logic incorrectly uses a logical OR operator instead of AND, allowing transactions to proceed even when the reserve ratio is outside the specified range. The validation should consider a ratio valid only when it is both above the minimum **AND** below the maximum, but the current implementation considers it valid if it is either above the minimum **OR** below the maximum.

The identified vulnerability resides in the `deposit_liquidity` function, as illustrated in figure 2.1, and in the `withdraw_liquidity` function.

```c
int valid = (compare_fractions(
    min_nominator,
    denominator,
    storage::reserve1,
    storage::reserve2
) <= 0) | (compare_fractions(
    max_nominator,
    denominator,
    storage::reserve1,
    storage::reserve2
) >= 0);
ifnot (valid) {
    excno = errors::reserves_ratio_failed;
}
```

**Figure 2.1:** The incorrect logical OR operator in the validation logic (dex/contracts/pool/include/jetton_based.fc#237–250)

The same logical operator error in the reserve ratio validation logic has also been found in the fee validation mechanism. However, in this case, the impact is lower because only admin operations can trigger the vulnerability, rather than arbitrary user transactions.

## Exploit Scenario
Alice submits a liquidity deposit transaction with specific min/max ratio parameters. Eve executes a swap that dramatically alters the pool’s ratio to be outside Alice’s intended range. Due to the logical OR error, Alice’s transaction still executes even though the ratio is now completely unfavorable.

## Recommendations
- **Short term:** Replace the logical OR operator (`|`) with a logical AND operator (`&`) in the reserve ratio validation logic to ensure that both conditions must be met for a valid range check.

- **Long term:** Implement comprehensive unit tests for the reserve ratio validation logic with various edge cases, including values at and beyond the boundaries of the specified ranges.

---
