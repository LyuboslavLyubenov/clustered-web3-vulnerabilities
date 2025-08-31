# Cluster -1102

**Rank:** #284  
**Count:** 27  

## Label
Inconsistent state validation and pricing logic enable attackers to exploit MEV, front-run yield updates, or manipulate reward distributions, leading to unfair gains, dilution, or protocol instability through predictable, low-effort attacks.

## Cluster Information
- **Total Findings:** 27

## Examples

### Example 1

**Auto Label:** Common vulnerability type: **Insufficient price validation and access controls enabling MEV-based manipulation and data integrity breaches**.  

**Original Text Preview:**

Calls to `setReserves` and `setPrice` creates opportunity for MEV, since users monitoring mempool can conduct transactions around calls to the functions either by frontrunning or backrunning to gain quick profit. The issue is further exarcebated in `setReserves` since there is no strictly defined upper bound through which price can move e.g price can move from somewhere as low as 1e6 to 1e18 which is the max as `PRICE_SCALING_FACTOR`.

From UsfPriceStorage.sol:

```solidity
    function setReserves(bytes32 _key, uint256 usfSupply, uint256 reserves) external onlyRole(SERVICE_ROLE) {
        if (_key == bytes32(0)) revert InvalidKey();
        if (usfSupply == 0) revert InvalidUsfSupply();
        if (reserves == 0) revert InvalidReserves();
        if (prices[_key].timestamp != 0) revert PriceAlreadySet(_key);

        uint256 computedPrice = _calculatePrice(usfSupply, reserves);
        uint256 lastPriceValue = lastPrice.price;
        if (lastPriceValue != 0) {
            // assumes only possible at initialization
            uint256 lowerBound = lastPriceValue - (lastPriceValue * lowerBoundPercentage / BOUND_PERCENTAGE_DENOMINATOR);
            if (computedPrice < lowerBound) {
                revert InvalidPrice(computedPrice, lowerBound);
            }
        }

        uint256 currentTime = block.timestamp;
@>      Price memory priceData =
            Price({price: computedPrice, usfSupply: usfSupply, reserves: reserves, timestamp: currentTime});

        prices[_key] = priceData;
        lastPrice = priceData;

        emit PriceSet(_key, computedPrice, usfSupply, reserves, currentTime);
    }
```

From FlpPriceStorage.sol:

```solidity
    function setPrice(bytes32 _key, uint256 _price) external onlyRole(SERVICE_ROLE) {
        if (_key == bytes32(0)) revert InvalidKey();
        if (_price == 0) revert InvalidPrice();
        if (prices[_key].timestamp != 0) revert PriceAlreadySet(_key);

        uint256 lastPriceValue = lastPrice.price;
        if (lastPriceValue != 0) {
            uint256 upperBound = lastPriceValue + (lastPriceValue * upperBoundPercentage / BOUND_PERCENTAGE_DENOMINATOR);
            uint256 lowerBound = lastPriceValue - (lastPriceValue * lowerBoundPercentage / BOUND_PERCENTAGE_DENOMINATOR);
            if (_price > upperBound || _price < lowerBound) {
                revert InvalidPriceRange(_price, lowerBound, upperBound);
            }
        }

        uint256 currentTime = block.timestamp;
@>      Price memory price = Price({price: _price, timestamp: currentTime});
        prices[_key] = price;
        lastPrice = price;

        emit PriceSet(_key, _price, currentTime);
    }
```

Recommendation is to use private RPC’s eliminating front-running and also introduce upper bound percentages in `setReserves` like is done in `setPrice`. The bound can then be capped to `PRICE_SCALING_FACTOR` if it exceeds it.

---
### Example 2

**Auto Label:** Common vulnerability type: **Insufficient price validation and access controls enabling MEV-based manipulation and data integrity breaches**.  

**Original Text Preview:**

## Summary

There is an intention to have a max deviation of price that could happen between updates and the [ScrvusdOracleV2.vy](https://github.com/CodeHawks-Contests/2025-03-curve/blob/198820f0c30d5080f75073243677ff716429dbfd/contracts/scrvusd/oracles/ScrvusdOracleV2.v) contract implements a price smoothing mechanism to do this verification and set up the new raw price, issue however is we have an with asymmetric behavior, cause the `_smoothed_price` function checks against `2 * max_change` in its conditional logic instead of `max_change`, creating inconsistent price movement constraints.

Specifically, when the new raw price is higher than the last price but less than 2 times the max change from the last price, i.e (`last_price + max_change < raw_price < last_price + 2*max_change`), the function allows the full price increase rather than limiting it. This asymmetry isn't done on the downside though, i.e when the raw price is less than `last_price - max_change`,cause the `unsafe_sub` operation would always return a very large sum in this case due to the underflow that would be more than `2 * max_change` and satisfy the condition to limit the price change.

## Vulnerability Details

First on the documentation side we can see how we expect the Oracle to correctly limit price change to `max_acceleration`:
https://docs.curve.fi/scrvusd/crosschain/oracle-v0/oracle/

> **scrvUSD Oracle**:
> Contract that contains information about the price of scrvUSD. It uses a `max_acceleration` parameter to limit the rate of price updates. The oracle includes a price_oracle method to ensure compatibility with other smart contracts, such as Stableswap implementations.

And in our case `max_acceleration` is the `max_change` as it's [set as the `max_price_increment`](https://github.com/CodeHawks-Contests/2025-03-curve/blob/198820f0c30d5080f75073243677ff716429dbfd/contracts/scrvusd/oracles/ScrvusdOracleV2.vy#L33-L35), that's used when calculating the `max_change`.

Going further into the maths, take a look at [ScrvusdOracleV2#smoothed_price()](https://github.com/CodeHawks-Contests/2025-03-curve/blob/198820f0c30d5080f75073243677ff716429dbfd/contracts/scrvusd/oracles/ScrvusdOracleV2.vy#L155-L165)

```python
def _smoothed_price(last_price: uint256, raw_price: uint256) -> uint256:
    # Ideally should be (max_price_increment / 10**18) ** (block.timestamp - self.last_update)
    # Using linear approximation to simplify calculations
    max_change: uint256 = (
        self.max_price_increment * (block.timestamp - self.last_update) * last_price // 10**18
    )
    # -max_change <= (raw_price - last_price) <= max_change
    if unsafe_sub(raw_price + max_change, last_price) > 2 * max_change:
        return last_price + max_change if raw_price > last_price else last_price - max_change
    return raw_price
```

As hinted under summary when smoothening the prices we want to restrict the change to `max_change` regardless of which side up/down.

Note that the value of `max*change` is relayed based on that of `max_price_increment` [which is `0.24 bps per block on Ethereum`](https://github.com/CodeHawks-Contests/2025-03-curve/blob/198820f0c30d5080f75073243677ff716429dbfd/contracts/scrvusd/oracles/ScrvusdOracleV2.vy#L98), amounting to a lump sum of over 1725 bps/per day _(+/- block mining)_, so for this report's sake if we consider looking at it from a day standpoint that would mean, we are allowing for a price change of over 3450 bps/per day on the up side which is unlike the correct restriction on the downside of `1725` bps.

And this happens because unlike the intended range of `-max_change <= (raw_price - last_price) <= max_change` we actually double it on the RHS of the equation due to the actual code implementation:

```python
if unsafe_sub(raw_price + max_change, last_price) > 2 * max_change:
```

Which creates distinct handling for price increases:

When `raw_price > last_price`:

- For outrageously large increases (`raw_price > last_price + 2*max_change`):
  - The condition is true, limiting the price to `last_price + max_change`
- For come large increases where the new raw price is bigger than our max change, i.e (`last_price + max_change < raw_price < last_price + 2*max_change`):
  - The condition is false, allowing the full `raw_price` without limitation
  - This contradicts the intended constraint of limiting to `max_change`

And we have inconsistent behavior between price increases and decreases, cause the above is not the case on the downside , cause for decreases, when `raw_price < last_price - max_change`, regardless of whether this raw_price is less than 2x the max_change or 10x the max_change, the `unsafe_sub` operation underflows, producing a very large value that always exceeds `2 * max_change`, forcing the price to be limited to `last_price - max_change`.

## Impact

First we have a broken functionality accross scope since the real range for the smoothening has been exaggerated 2X for increases and would then mean that the security measure used to maintain price stabilty and stop abrupt price changes in the upside is broken, allowing the main solution to the problem of MEV extraction or price manipulation to be relaxed and attackers to craft their way to extract value, since they can predict this assymetry based on what's happening on mainnet, i.e in our shared hypothetical case we allow for an increase of around `35%` on the up side.

> NB: Whereas `_smoothed_price` is a `view` modified function, it's used all round when updating prices to limit these abrupt changes with the flow to get to it being via: `update_price()` -> `self._price_v0() && self._price_v1() && self._price_v2()` -> `_smoothed_price()`.

## Tools Used

Manual review

## Recommendations

Revise the conditional logic to properly enforce the intended constraint of limiting price changes to `±max_change` and not `±2max_change`

```diff
@view
def _smoothed_price(last_price: uint256, raw_price: uint256) -> uint256:
    # Ideally should be (max_price_increment / 10**18) ** (block.timestamp - self.last_update)
    # Using linear approximation to simplify calculations
    max_change: uint256 = (
        self.max_price_increment * (block.timestamp - self.last_update) * last_price // 10**18
    )
    # -max_change <= (raw_price - last_price) <= max_change
-    if unsafe_sub(raw_price + max_change, last_price) > 2 * max_change:
+    if unsafe_sub(raw_price + max_change, last_price) > max_change:
        return last_price + max_change if raw_price > last_price else last_price - max_change
    return raw_price

```

## M Current `MAX_V2_DURATION` curbs price growth projection at ~ 10% earlier than intended

### Summary

The `ScrvusdOracleV2` contract incorrectly calculates the MAX*V2_DURATION constant, which is intended to represent 4 years in terms of weekly periods. The current implementation uses 192 weeks (4 * 12 \_ 4), which assumes 48 weeks per year instead of the standard 52 weeks. This discrepancy results in premature limitation of price growth projection, affecting the accuracy of long-term price calculations.

### Vulnerability Details

Take a look at [ScrvusdOracleV2.vy#L57](https://github.com/CodeHawks-Contests/2025-03-curve/blob/198820f0c30d5080f75073243677ff716429dbfd/contracts/scrvusd/oracles/ScrvusdOracleV2.vy#L57)

```
MAX_V2_DURATION: constant(uint256) = 4 * 12 * 4  # 4 years
```

The comment indicates that this constant is meant to represent 4 years, but the calculation uses 4 _ 12 _ 4 = 192 weeks. A standard year has approximately 52 weeks (365 days / 7 days per week), so 4 years should be represented as 4 \* 52 = 208 weeks.

This constant is used in the `_obtain_price_params` function to limit how far into the future the price projection can go:

```python
number_of_periods: uint256 = min(
    (parameters_ts - params.last_profit_update) // period,
    self.max_v2_duration,
)
```

The `max_v2_duration` value is set in the constructor to "half a year" (4 \* 6 = 24 weeks), but can be updated through the `set_max_v2_duration` function, which enforces that the new value doesn't exceed `MAX_V2_DURATION`.

### Impact

The incorrect calculation of `MAX_V2_DURATION` means that price growth projection is limited to 192 weeks instead of the intended 208 weeks (4 years) when we inted to use the real max and this results in:

- Premature limitation of price growth projection by approximately 16 weeks ~ 10% of the whole duration
- Most crucially is the fact that this would cause for a deviation of the growth pattern on the destination chain than what is on mainnet which is a security invariant for this protocol.

### Tools Used

Manual review

### Recommendations

```diff
- MAX_V2_DURATION: constant(uint256) = 4 * 12 * 4  # 4 years
+ MAX_V2_DURATION: constant(uint256) = 4 * 52  # 4 years (52 weeks per year)
```

In the same light the current v2 duration if intended to be 6 months should be set to the below:

https://github.com/CodeHawks-Contests/2025-03-curve/blob/198820f0c30d5080f75073243677ff716429dbfd/contracts/scrvusd/oracles/ScrvusdOracleV2.vy#L75-L105

```diff
@deploy
def __init__(_initial_price: uint256):
#snip
    # 2 * 10 ** 12 is equivalent to
    #   1) 0.02 bps per second or 0.24 bps per block on Ethereum
    #   2) linearly approximated to max 63% APY
    self.max_price_increment = 2 * 10**12
-    self.max_v2_duration = 4 * 6  # half a year
+    self.max_v2_duration = 26  # half a year

    access_control.__init__()
    access_control._set_role_admin(PRICE_PARAMETERS_VERIFIER, access_control.DEFAULT_ADMIN_ROLE)
    access_control._set_role_admin(UNLOCK_TIME_VERIFIER, access_control.DEFAULT_ADMIN_ROLE)


```

## H Using `last_profit_update` as a timestamp surrogate causes price calculation divergence and would stall period of profit evolution which breaks accounting

### Summary

The `ScrvusdVerifierV1.sol` contract contains a critical design flaw in how timestamps are used across different verification methods. In `verifyScrvusdByStateRoot()`, the function uses `last_profit_update` as a timestamp surrogate for price calculations, whereas the `verifyScrvusdByBlockHash()` function uses the actual block timestamp. This inconsistency creates a divergence in price calculations between these two verification paths, as `last_profit_update` is typically lagging behind the current block timestamp, resulting in inconsistent oracle prices based on which verification method is used.

### Vulnerability Details

Take a look at [ScrvusdVerifierV1::verifyScrvusdByStateRoot()](https://github.com/code-423n4/2025-03-curve/blob/main/contracts/scrvusd/verifiers/ScrvusdVerifierV1.sol#L69-L82)

```solidity
function verifyScrvusdByStateRoot(
    uint256 _block_number,
    bytes memory _proof_rlp
) external returns (uint256) {
    bytes32 state_root = IBlockHashOracle(BLOCK_HASH_ORACLE).get_state_root(_block_number);

    uint256[PARAM_CNT] memory params = _extractParametersFromProof(state_root, _proof_rlp);
    // Use last_profit_update as the timestamp surrogate
    return _updatePrice(params, params[5], _block_number);
}
```

The comment clearly indicates that `last_profit_update` (params[5]) is being used as a "timestamp surrogate." However, this creates an inconsistency when compared to the other verification method:

```solidity
function verifyScrvusdByBlockHash(
    bytes memory _block_header_rlp,
    bytes memory _proof_rlp
) external returns (uint256) {
    Verifier.BlockHeader memory block_header = Verifier.parseBlockHeader(_block_header_rlp);
    require(block_header.hash != bytes32(0), "Invalid blockhash");
    require(
        block_header.hash == IBlockHashOracle(BLOCK_HASH_ORACLE).get_block_hash(block_header.number),
        "Blockhash mismatch"
    );

    uint256[PARAM_CNT] memory params = _extractParametersFromProof(block_header.stateRootHash, _proof_rlp);
    return _updatePrice(params, block_header.timestamp, block_header.number);
}
```

This method uses the actual `block_header.timestamp` instead of the `last_profit_update` value.

The issue becomes critical when we examine how `last_profit_update` is updated in the scrvusd system.

To go into more details, in scrvusd's `VaultV3.vy`, this value is only updated during profit processing:

https://etherscan.io/address/0x0655977FEb2f289A4aB78af67BAB0d17aAb84367#code#L1301

```python
## Update the last profitable report timestamp.
self.last_profit_update = block.timestamp
```

And this update only occurs in the `_process_report` function, which is exclusively called by accounts with the `REPORTING_MANAGER` role:

https://etherscan.io/address/0x0655977FEb2f289A4aB78af67BAB0d17aAb84367#code#L1638

```python
@external
@nonreentrant("lock")
def process_report(strategy: address) -> (uint256, uint256):
    """
    @notice Process the report of a strategy.
    @param strategy The strategy to process the report for.
    @return The gain and loss of the strategy.
    """
    self._enforce_role(msg.sender, Roles.REPORTING_MANAGER)
    return self._process_report(strategy)
```

Since `last_profit_update` is only updated when: 1) there's a profit report and 2) only by specific managers, we then have a significant time lag between the current block timestamp and the `last_profit_update` value in most blocks.

This would then mean that the timestamp used in ScrvusdVerifierV1's `verifyScrvusdByStateRoot` will always be older than the current block timestamp.

Now back in the oracle, i.e `ScrvusdOracleV2.vy`, the contract's price calculation is highly dependent on this timestamp parameter:

```python
@external
def update_price(
    _parameters: uint256[ALL_PARAM_CNT], _ts: uint256, _block_number: uint256
) -> uint256:
    # ...
    ts: uint256 = self.price_params_ts
    current_price: uint256 = self._raw_price(ts, ts)
    # ...
    self.price_params_ts = _ts

    new_price: uint256 = self._raw_price(_ts, _ts)
    # ...
```

When `_ts` is set to the outdated `last_profit_update` instead of the current timestamp, it leads to divergent price calculations between the two verification methods.

Cause technically we could have two consequent updates where we then pass in an older `ts`:

- Assume we verify a new block N using `verifyScrvusdByBlockHash`
- We then verify the next block N+1 using `verifyScrvusdByStateRoot`.
- Since the `last_profit_update` could be older than the block timestamp of block N, the price calculation will be different between the two verification methods and we actually would have a reversal logic on the smoothening in the oracle implementation.

This is because in the oracle:

- The `_raw_price` calculates the price based on these timestamps:

```python
def _raw_price(ts: uint256, parameters_ts: uint256) -> uint256:
    parameters: PriceParams = self._obtain_price_params(parameters_ts)
    return self._total_assets(parameters) * 10**18 // self._total_supply(parameters, ts)
```

- The `_obtain_price_params` function however is where the timestamp critically affects calculations:

```python
def _obtain_price_params(parameters_ts: uint256) -> PriceParams:
    # ...
    if params.last_profit_update + period >= parameters_ts:
        return params

    number_of_periods: uint256 = min(
        (parameters_ts - params.last_profit_update) // period,
        self.max_v2_duration,
    )
    # ... calculations based on number_of_periods ...
```

- The timestamp also affects calculations in the `_total_supply` function:

```python
def _total_supply(p: PriceParams, ts: uint256) -> uint256:
    return p.total_supply - self._unlocked_shares(
        p.full_profit_unlock_date,
        p.profit_unlocking_rate,
        p.last_profit_update,
        p.balance_of_self,
        ts,  # block.timestamp
    )
```

Cause this would mean we would have a return of unlocked shares being larger than it should be:
https://github.com/CodeHawks-Contests/2025-03-curve/blob/198820f0c30d5080f75073243677ff716429dbfd/contracts/scrvusd/oracles/ScrvusdOracleV2.vy#L189-L215

```python
@view
def _unlocked_shares(
    full_profit_unlock_date: uint256,
    profit_unlocking_rate: uint256,
    last_profit_update: uint256,
    balance_of_self: uint256,
    ts: uint256,
) -> uint256:
## ..snip
    unlocked_shares: uint256 = 0
    if full_profit_unlock_date > ts:
        # If we have not fully unlocked, we need to calculate how much has been.
        unlocked_shares = profit_unlocking_rate * (ts - last_profit_update) // MAX_BPS_EXTENDED
## @audit for the above since `ts` is lower than it should be, unlocked_shares would be higher than it should be
    elif full_profit_unlock_date != 0:
        # All shares have been unlocked
        unlocked_shares = balance_of_self

    return unlocked_shares


```

Which would then mean that on the basis of this alone, our total supply would be less, since it's calculated as `p.total_supply - self._unlocked_shares()`, [and this goes ahead to flaw the raw price calculation and have the price higher than it should actually be](https://github.com/CodeHawks-Contests/2025-03-curve/blob/198820f0c30d5080f75073243677ff716429dbfd/contracts/scrvusd/oracles/ScrvusdOracleV2.vy#L286-L293).

So going back the two scenarios:

1. **In scenario with Block N using `verifyScrvusdByBlockHash`**:

   - The oracle receives the actual block timestamp (let's say 100)
   - It calculates `number_of_periods` based on `(100 - last_profit_update) // period`
   - This provides a relatively recent and accurate price calculation

2. **In scenario with Block N+1 using `verifyScrvusdByStateRoot`**:
   - The oracle receives `last_profit_update` as the timestamp (let's say 70, which is older)
   - It calculates `number_of_periods` based on `(70 - last_profit_update) // period`
   - Since 70 is the actual value of `last_profit_update`, _that should be passed in the verification circa block N_ so this calculation results in 0 periods
   - This effectively means no profit evolution is calculated

The core issue is that `_obtain_price_params` relies on the difference between `parameters_ts` and `params.last_profit_update` to calculate how many periods of profit evolution should be applied. When `parameters_ts` equals `params.last_profit_update` (which happens when passing `last_profit_update` as the timestamp), the function sees 0 periods and doesn't evolve the parameters at all.

### Impact

This timestamp inconsistency creates a critical vulnerability in the price oracle system with several severe consequences:

When consequently using the two different verification methods, prices calculated will diverge over time due to different timestamp handling:

- `verifyScrvusdByBlockHash` will properly account for time passage
- `verifyScrvusdByStateRoot` will use stale timing data, causing profit evolution calculations to stagnate and be broken.

The calculation of `number_of_periods` in `_obtain_price_params` will be 0 when using `last_profit_update`, since:

```python
## This calculation returns 0 when parameters_ts = last_profit_update
number_of_periods: uint256 = (parameters_ts - params.last_profit_update) // period
```

The `_unlocked_shares` that's used when getting the total supply depends on an accurate timestamp to calculate profit unlocking:

```python

@view
def _total_supply(p: PriceParams, ts: uint256) -> uint256:
 # Need to account for the shares issued to the vault that have unlocked.
 return p.total_supply - self._unlocked_shares(
     p.full_profit_unlock_date,
     p.profit_unlocking_rate,
     p.last_profit_update,
     p.balance_of_self,
     ts,  # block.timestamp
 )

def _unlocked_shares(..., ts: uint256) -> uint256:
    if full_profit_unlock_date > ts:
        unlocked_shares = profit_unlocking_rate * (ts - last_profit_update) // MAX_BPS_EXTENDED
```

Note that this value from the total supply is used [when setting the raw price](https://github.com/CodeHawks-Contests/2025-03-curve/blob/198820f0c30d5080f75073243677ff716429dbfd/contracts/scrvusd/oracles/ScrvusdOracleV2.vy#L286-L292):

```python
def _raw_price(ts: uint256, parameters_ts: uint256) -> uint256:
    """
    @notice Price replication from scrvUSD vault
    """
    parameters: PriceParams = self._obtain_price_params(parameters_ts)
    return self._total_assets(parameters) * 10**18 // self._total_supply(parameters, ts)

```

Since we have a wrong denominator and a wrong numerator, among all other impacts we will have a wrong price calculation, which will be set in that update:

https://github.com/CodeHawks-Contests/2025-03-curve/blob/198820f0c30d5080f75073243677ff716429dbfd/contracts/scrvusd/oracles/ScrvusdOracleV2.vy#L294-L331

```python
@external
def update_price(
    _parameters: uint256[ALL_PARAM_CNT], _ts: uint256, _block_number: uint256
) -> uint256:
    #snip
    ts: uint256 = self.price_params_ts
    current_price: uint256 = self._raw_price(ts, ts)
    self.price_params = PriceParams(
        total_debt=_parameters[0],
        total_idle=_parameters[1],
        total_supply=_parameters[2],
        full_profit_unlock_date=_parameters[3],
        profit_unlocking_rate=_parameters[4],
        last_profit_update=_parameters[5],
        balance_of_self=_parameters[6],
    )
    self.price_params_ts = _ts

|>    new_price: uint256 = self._raw_price(_ts, _ts)
    log PriceUpdate(new_price, _ts, _block_number)
    if new_price > current_price:
        return (new_price - current_price) * 10**18 // current_price
    return (current_price - new_price) * 10**18 // current_price

```

The practical implications include unreliable price data for any protocols relying on this oracle, mispriced assets, incorrect liquidations, and arbitrage opportunities that harm liquidity providers, etc all which fall under the main problem attempted to be solved by the system:

> **Problem**: It is a hard problem to guarantee the correctness of the value provided by the oracle. If not precise enough, this can
> lead to MEV in the liquidity pool, at a loss for the liquidity providers. Even worse, if someone is able to manipulate this rate, it can lead to the pool being drained from one side.

### Coded POC

Create a new file under the `tests/scrvusd/verifier/unitary` directory called `test_timestamp_broken.py`:

```python
import pytest
import rlp
import boa

from scripts.scrvusd.proof import serialize_proofs
from tests.conftest import WEEK
from tests.scrvusd.verifier.conftest import MAX_BPS_EXTENDED
from tests.shared.verifier import get_block_and_proofs


@pytest.fixture(scope="module")
def scrvusd_slot_values(scrvusd, crvusd, admin, anne):
    deposit = 10**18
    with boa.env.prank(anne):
        crvusd._mint_for_testing(anne, deposit)
        crvusd.approve(scrvusd, deposit)
        scrvusd.deposit(deposit, anne)
        # New scrvusd parameters:
        #   scrvusd.total_idle = deposit,
        #   scrvusd.total_supply = deposit.

    rewards = 10**17
    with boa.env.prank(admin):
        crvusd._mint_for_testing(scrvusd, rewards)
        scrvusd.process_report(scrvusd)
        # Minted `rewards` shares to scrvusd, because price is still == 1.

    # Record the initial values
    last_profit_update = boa.env.evm.patch.timestamp

    # Travel forward in time to create a significant gap between current timestamp and last_profit_update
    boa.env.time_travel(seconds=3*86400, block_delta=3*12)  # 3 days forward

    return {
        "total_debt": 0,
        "total_idle": deposit + rewards,
        "total_supply": deposit + rewards,
        "full_profit_unlock_date": last_profit_update + WEEK,
        "profit_unlocking_rate": rewards * MAX_BPS_EXTENDED // WEEK,
        "last_profit_update": last_profit_update,
        "balance_of_self": rewards,
    }


def test_using_last_profit_update_as_timestamp_surrogate_is_broken(
    verifier, soracle_price_slots, soracle, boracle, scrvusd, scrvusd_slot_values
):
    """
    Test demonstrates how using last_profit_update as a timestamp surrogate in verifyScrvusdByStateRoot
    leads to divergent price calculations compared to verifyScrvusdByBlockHash which uses current timestamp.

    """
    # Get current and previous timestamps for analysis
    current_timestamp = boa.env.evm.patch.timestamp
    last_profit_update = scrvusd_slot_values["last_profit_update"]

    # CASE 1: Using verifyScrvusdByBlockHash (which uses current timestamp)
    # --------------------------------------------------------------------------

    block_header, proofs = get_block_and_proofs([(scrvusd, soracle_price_slots)])
    boracle._set_block_hash(block_header.block_number, block_header.hash)

    # Execute verification using blockHash method
    tx1 = verifier.verifyScrvusdByBlockHash(
        rlp.encode(block_header),
        serialize_proofs(proofs[0]),
    )

    # Record the timestamp used after blockHash verification
    blockhash_timestamp = soracle._storage.price_params_ts.get()

    # Save the value after blockHash call to compare later
    blockhash_block_number = soracle.last_block_number

    # CASE 2: Using verifyScrvusdByStateRoot (which uses last_profit_update as timestamp)
    # -----------------------------------------------------------------------------------

    # Set up new state root verification
    block_header, proofs = get_block_and_proofs([(scrvusd, soracle_price_slots)])
    boracle._set_state_root(block_header.block_number, block_header.state_root)

    # Execute verification using stateRoot method
    tx2 = verifier.verifyScrvusdByStateRoot(
        block_header.block_number,
        serialize_proofs(proofs[0]),
    )

    # Record the timestamp used after stateRoot verification
    stateroot_timestamp = soracle._storage.price_params_ts.get()

    # ANALYSIS: Compare the results
    # ------------------------------------

    # Verify the timestamps used are different
    assert blockhash_timestamp != stateroot_timestamp, "Timestamps should be different between methods"

    # Calculate periods that would be applied in each method
    daily_period = 86400  # 1 day in seconds
    periods_blockHash = (blockhash_timestamp - last_profit_update) // daily_period
    periods_stateRoot = (stateroot_timestamp - last_profit_update) // daily_period

    print(f"\nProfit evolution periods with blockHash method: {periods_blockHash}")
    print(f"Profit evolution periods with stateRoot method: {periods_stateRoot}")

    assert abs(stateroot_timestamp - last_profit_update) == 0, "StateRoot should use last_profit_update as timestamp"
    assert blockhash_timestamp != stateroot_timestamp, "Timestamps should differ between methods"

    # Show impact on profit evolution since periods would be zero when verification is done with stateRoot
    assert periods_blockHash != 0
    assert periods_stateRoot == 0

```

Run test with:

```bash

python -m pytest tests/scrvusd/verifier/unitary/test_timestamp_broken.py -v
```

Log output

```stdout
=================================================================================================================================================================================================================== test session starts ===================================================================================================================================================================================================================
platform darwin -- Python 3.12.6, pytest-8.3.4, pluggy-1.5.0 --..//codebases/2025-03-curve/.venv/bin/python
cachedir: .pytest_cache
hypothesis profile 'default' -> database=DirectoryBasedExampleDatabase(PosixPath('/Users/abdullahisuleimanaliyu/Desktop/codebases/2025-03-curve/.hypothesis/examples'))
rootdir:..//codebases/2025-03-curve
configfile: pyproject.toml
plugins: titanoboa-0.2.5, cov-6.0.0, hypothesis-6.122.3
collected 1 item

tests/scrvusd/verifier/unitary/test_timestamp_broken.py::test_using_last_profit_update_as_timestamp_surrogate_is_broken PASSED                                                                                                                                                                                                                                                                                                                      [100%]

```

### Tools Used

Manual review

### Recommendations

Properly account for the current block.timestamp even during verifications using `verifyScrvusdByStateRoot` or do away with this verification method and only use `verifyScrvusdByBlockHash`.

## H Price smoothing only working till `max_change` leads to price deviation on the destination chain post smooth

### Summary

The `ScrvusdOracleV2` contract implements a price smoothing mechanism with a `max_change` limitation that prevents an abrupt price changes for the three `lastprice[3]` or `price_v0()|price_v1()|price_v2()`, which has been [relayed in the walkthrough video that it's needed in the case where the keeper goes off for a while](https://www.youtube.com/live/oJ1IiW3Abr4?si=bCa7m_prpNWSwKDx), this max change however stops the price oracle from accurately reflecting verified mainnet prices on destination chains since it creates a hard cap on how much the price can change in a single update, regardless of the verified data from Ethereum.

As a result, when significant price movements occur on the source chain, the destination chain's price becomes immediately out of sync post the smoothing duration, breaking the critical security invariant of cross-chain price consistency and creating exploitable arbitrage opportunities.

### Vulnerability Details

Take a look at [ScrvusdOracleV2#smoothed_price()](https://github.com/CodeHawks-Contests/2025-03-curve/blob/198820f0c30d5080f75073243677ff716429dbfd/contracts/scrvusd/oracles/ScrvusdOracleV2.vy#L154-L166)

```python
@view
def _smoothed_price(last_price: uint256, raw_price: uint256) -> uint256:
    # Ideally should be (max_price_increment / 10**18) ** (block.timestamp - self.last_update)
    # Using linear approximation to simplify calculations
    max_change: uint256 = (
        self.max_price_increment * (block.timestamp - self.last_update) * last_price // 10**18
    )
    # -max_change <= (raw_price - last_price) <= max_change
    if unsafe_sub(raw_price + max_change, last_price) > 2 * max_change:
        return last_price + max_change if raw_price > last_price else last_price - max_change
    return raw_price
```

The critical issue is in the conditional logic that limits price changes to `max_change`. When the difference between `raw_price` and `last_price` exceeds this limit, the function returns a capped value instead of the actual verified price:

```python
return last_price + max_change if raw_price > last_price else last_price - max_change
```

This value is then what the three [price_v0()|price_v1()|price_v2()](https://github.com/CodeHawks-Contests/2025-03-curve/blob/198820f0c30d5080f75073243677ff716429dbfd/contracts/scrvusd/oracles/ScrvusdOracleV2.vy#L167-L187) are limited to:

```python
@view
def _price_v0() -> uint256:
    return self._smoothed_price(
        self.last_prices[0],
        self._raw_price(self.price_params_ts, self.price_params.last_profit_update),
    )


@view
def _price_v1() -> uint256:
    return self._smoothed_price(
        self.last_prices[1], self._raw_price(block.timestamp, self.price_params_ts)
    )


@view
def _price_v2() -> uint256:
    return self._smoothed_price(
        self.last_prices[2], self._raw_price(block.timestamp, block.timestamp)
    )

```

But creates a permanent deviation from the true price on Ethereum that cannot be recovered from, even with subsequent updates. Each update is limited by the same `max_change` constraint, causing the destination chain to perpetually lag behind the source chain when significant price movements occur.

For example:

1. Current price on destination chain: 100
2. Maximum allowed change (max_change): 10
3. True price on Ethereum (verified): 70
4. First update: Price can only move to 90 (100 - 10)
5. Second update: Even with the same verified price of 70, the price can only move to 80 (90 - 10)
6. Third update assume verified price is 60: Price moves to 70, still not matching Ethereum.

**In the above would be key to note that out of all three updates only one is more than the max_change, but the price oracle still relays wrong prices for all updates**

So, if the Ethereum price continues to change during this time, the destination chain will never catch up. This creates a persistent price discrepancy that can be exploited.

The `price_v2()` function, which should theoretically provide the most accurate price approximation(i.e theoretically, `price_v2() == raw_price` post the smoothening duration), is also limited by the `max_change` constraint.

```python
@view
def _price_v2() -> uint256:
    return self._smoothed_price(
        self.last_prices[2], self._raw_price(block.timestamp, block.timestamp)
    )

@view
@external
def raw_price(
    _i: uint256 = 0, _ts: uint256 = block.timestamp, _parameters_ts: uint256 = block.timestamp
) -> uint256:
    p: uint256 = self._raw_price(_ts, _parameters_ts)
    return p if _i == 0 else 10**36 // p


```

Which is because it uses the current timestamp just as the parameters of `_raw_price()`, the `max_change` limitation prevents it from accurately reflecting the true price.

> NB: This seems to have been put as another security measure [against ingesting an incorrect block hash so we dont have an outrageous price](https://github.com/CodeHawks-Contests/2025-03-curve/blob/198820f0c30d5080f75073243677ff716429dbfd/contracts/scrvusd/oracles/ScrvusdOracleV2.vy#L307), however this is not sufficient as [up till past the max change is allowed to be stored](https://github.com/CodeHawks-Contests/2025-03-curve/blob/198820f0c30d5080f75073243677ff716429dbfd/contracts/scrvusd/oracles/ScrvusdOracleV2.vy#L162) and can't be removed.

### Impact

> TLDR: This breaks the security invariant of not having deviated prices on dest chain.

First, this would cause for a broken price pattern in the case an incorrect block hash is ingested, cause where as we allow for it to be immediately updated by the prover, this wrong data has already been ingested into the `lastPrices` and can not be removed:

https://github.com/CodeHawks-Contests/2025-03-curve/blob/198820f0c30d5080f75073243677ff716429dbfd/contracts/scrvusd/oracles/ScrvusdOracleV2.vy#L294-L332

```python
@external
def update_price(
    _parameters: uint256[ALL_PARAM_CNT], _ts: uint256, _block_number: uint256
) -> uint256:
    access_control._check_role(PRICE_PARAMETERS_VERIFIER, msg.sender)
    # Allowing same block updates for fixing bad blockhash provided (if possible)
|>    assert self.last_block_number <= _block_number, "Outdated"
    self.last_block_number = _block_number

|>    self.last_prices = [self._price_v0(), self._price_v1(), self._price_v2()]
    self.last_update = block.timestamp

## ..snip

```

Also the fact that even post smooth we end up with wrong price intself means arbitrage opportunities are back on the table circa MEV issues and what not.

### Tools Used

Manual review

### Recommendations

Remove the hard cap on price changes for cryptographically verified updates from Ethereum. Since these updates are already to be proven legitimate via blockhash or state root verification, they should be trusted and applied directly.

The smoothening in itself is already a sufficient measure to prevent sharp price changes.

Alternatively have an admin backed method that can directly update the `lastprices[3]` on the destination chain.

## M Having a fixed inaccurate initial price value enables price manipulation and breaks cross-chain price equality for a while

### Summary

The `ScrvusdOracleV2` contract initializes the price at a fixed value of 1 (10^18) regardless of the actual scrvUSD price on Ethereum mainnet at deployment time. This creates a significant discrepancy between the initial oracle price on the destination chain and the actual price on Ethereum, enabling price manipulation and breaking the fundamental security invariant of maintaining consistent prices across chains.

### Vulnerability Details

_From [ScrvusdOracleV2::**init**()](https://github.com/CodeHawks-Contests/2025-03-curve/blob/198820f0c30d5080f75073243677ff716429dbfd/contracts/scrvusd/oracles/ScrvusdOracleV2.vy#L74-L105)_

```python
@deploy
def __init__(_initial_price: uint256):
    """
    @param _initial_price Initial price of asset per share (10**18)
    """
    self.last_prices = [_initial_price, _initial_price, _initial_price]
    self.last_update = block.timestamp

    # initial raw_price is 1
    self.profit_max_unlock_time = 7 * 86400  # Week by default
    self.price_params = PriceParams(
        total_debt=0,
        total_idle=1,
        total_supply=1,
        full_profit_unlock_date=0,
        profit_unlocking_rate=0,
        last_profit_update=0,
        balance_of_self=0,
    )
```

While the `init` accepts an `_initial_price` parameter, the comment "initial raw_price is 1" and the initialization of `price_params` with `total_idle=1` and `total_supply=1` effectively sets the raw price to 1, regardless of the actual scrvUSD price on Ethereum at the time of deployment.

According to the project README, the primary purpose of this oracle is to ensure consistent pricing of scrvUSD across chains:

> "To address this problem, we opted to have secondary scrvUSD markets on all chains where scrvUSD can be redeemed. Since the price of the asset is not stable, we cannot use a 'simple' stableswap-ng pool as the price of the asset would go up as the yield accrues."

The oracle's role is to "fetch scrvUSD vault parameters from Ethereum, and provide them on other chains, with the goal of being able to compute the growth rate in a safe (non-manipulable) and precise (no losses due to approximation) way."

But this has then been broken [and we then have to wait until a prover sends in a valid block hash or stateroot to then update the price](https://github.com/CodeHawks-Contests/2025-03-curve/blob/198820f0c30d5080f75073243677ff716429dbfd/contracts/scrvusd/verifiers/ScrvusdVerifierV1.sol#L52-L81), this is unlike what's done in the case of the profit_max_unlock time, [since it's set correctly to 7 days](https://github.com/CodeHawks-Contests/2025-03-curve/blob/198820f0c30d5080f75073243677ff716429dbfd/contracts/scrvusd/oracles/ScrvusdOracleV2.vy#L84).

### Impact

Since the actual `scrvUSD` price on Ethereum is not exactly 1 at deployment time (which is highly likely as scrvUSD accrues yield), there will be an immediate price discrepancy between chains, which would then allow for arbitrage opportunities where attackers can exploit the price difference between chains, potentially draining liquidity from pools on the destination chain.

The impact is particularly severe cause protocol intends to deploy on any/all EVM known chains, so now post the initial deployment period on any new EVM chain that's to be supported by the and before the first price update occurs, we have a window where the price on the destination chain is completely disconnected from the actual price on Ethereum and this all depends on how long it takes the prover to send in the update.

### Tools Used

Manual review

### Recommendations

The initialization should use the actual `scrvUSD` price from Ethereum at deployment time which the `deployer` should in this case be trusted and correctly provide, this can also be easily done by passing in a `params` array during deployment.

Alternatively, pricing should not be updated during init, and instead there should be a flag that allows for querying prices only after the first price update has occurred, in the price getters.

---
### Example 3

**Auto Label:** Common vulnerability type: **Insufficient price validation and access controls enabling MEV-based manipulation and data integrity breaches**.  

**Original Text Preview:**

## Summary

The NatSpec documentation for the `setHousePrice` function incorrectly states that it allows the owner to set the house price for a token, while the implementation restricts access to accounts with the `onlyOracle` role. This discrepancy creates ambiguity regarding who is authorized to update house prices. Additionally, the function does not verify whether the token exists before updating its price.

## Vulnerability Details

* NatSpec vs. Implementation Mismatch:
  The NatSpec comment describes the function as being callable by the owner:
  ```solidity
  /**
   * @notice Allows the owner to set the house price for a token
   * @param _tokenId The ID of the RAAC token
   * @param _amount The price to set for the house in USD
   */
  ```
  However, the function is defined with the `onlyOracle` modifier:
  ```solidity
  function setHousePrice(
      uint256 _tokenId,
      uint256 _amount
  ) external onlyOracle { ... }
  ```
  This mismatch leads to confusion about whether the function is intended to be restricted to the owner or to an oracle.

## Impact

* Access Control Confusion:\
  Stakeholders might assume that the owner is responsible for setting house prices when, in fact, only an account with the `onlyOracle` role can perform this operation. This can lead to misconfigurations and potential security gaps if the intended access control is not enforced.

## Tools Used

* Manual code review

## Recommended Mitigation

* Clarify the NatSpec Documentation:\
  If the intended caller is the oracle, update the NatSpec comment to reflect that the function allows the oracle (and not the owner) to set the house price:
  ```solidity
  /**
   * @notice Allows the oracle to set the house price for a token
   * @param _tokenId The ID of the RAAC token
   * @param _amount The price to set for the house in USD
   *
   * Updates the timestamp for each token individually.
   */
  ```

* Review and Adjust Access Control:\
  Determine the intended role:
  * If the owner should set the house price, change the modifier from `onlyOracle` to `onlyOwner`.
  * If the oracle is intended to set the price, ensure that the documentation and role management are updated accordingly.

---
