# Cluster -1312

**Rank:** #443  
**Count:** 8  

## Label
High gas cost and systemic failure due to repeated external calls and oracle dependency, leading to scalability issues and widespread protocol collapse under failure conditions.

## Cluster Information
- **Total Findings:** 8

## Examples

### Example 1

**Auto Label:** Premature exits and flawed validation lead to missed callbacks, cascading failures, and unauthorized fee drainage, compromising oracle integrity and protocol reliability.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-02-perennial-v2-3-judging/issues/6 

## Found by 
bin2chen, panprog
## Summary

Each market action requests a new oracle version which must be commited by the keepers. However, if keepers are unable to commit requested version's price (for example, no price is available at the time interval, network or keepers are down), then after a certain timeout this oracle version will be commited as invalid, using the previous valid version's price.

The issue is that when this expired oracle version is used by the market (using `oracle.at`), the version returned will be valid (`valid = true`), because oracle returns version as invalid only if `price = 0`, but the `commit` function sets the previous version's price for these, thus it's not 0.

This leads to market using invalid versions as if they're valid, keeping the orders (instead of invalidating them), which is a broken core functionality and a security risk for the protocol.

## Vulnerability Detail

When requested oracle version is commited, but is expired (commited after a certain timeout), the price of the previous valid version is set to this expired oracle version:
```solidity
function _commitRequested(OracleVersion memory version) private returns (bool) {
    if (block.timestamp <= (next() + timeout)) {
        if (!version.valid) revert KeeperOracleInvalidPriceError();
        _prices[version.timestamp] = version.price;
    } else {
        // @audit previous valid version's price is set for expired version
        _prices[version.timestamp] = _prices[_global.latestVersion]; 
    }
    _global.latestIndex++;
    return true;
}
```

Later, `Market._processOrderGlobal` reads the oracle version using the `oracle.at`, invalidating the order if the version is invalid:
```solidity
function _processOrderGlobal(
    Context memory context,
    SettlementContext memory settlementContext,
    uint256 newOrderId,
    Order memory newOrder
) private {
    OracleVersion memory oracleVersion = oracle.at(newOrder.timestamp);

    context.pending.global.sub(newOrder);
    if (!oracleVersion.valid) newOrder.invalidate();
```

However, expired oracle version will return `valid = true`, because this flag is only set to `false` if `price = 0`:
```solidity
function at(uint256 timestamp) public view returns (OracleVersion memory oracleVersion) {
    (oracleVersion.timestamp, oracleVersion.price) = (timestamp, _prices[timestamp]);
    oracleVersion.valid = !oracleVersion.price.isZero(); // @audit <<< valid = false only if price = 0
}
```

This means that `_processOrderGlobal` will treat this expired oracle version as valid and won't invalidate the order.

## Impact

Market uses invalid (expired) oracle versions as if they're valid, keeping the orders (instead of invalidating them), which is a broken core functionality and a security risk for the protocol.

## Code Snippet

`KeeperOracle._commitRequested` sets `_prices` to the last valid version's price for expired versions:
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L153-L162

`Market._processOrderGlobal` reads the oracle version using the `oracle.at`, invalidating the order if the version is invalid:
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L604-L613

`KeeperOracle.at` returns `valid = false` only if `price = 0`, but since expired version has valid price, it will be returned as a valid version:
https://github.com/sherlock-audit/2024-02-perennial-v2-3/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L109-L112

## Tool used

Manual Review

## Recommendation

Add validity map along with the price map to `KeeperOracle` when recording commited price.



## Discussion

**nevillehuang**

@arjun-io @panprog @bin2chen66 For the current supported tokens in READ.ME, I think medium severity remains appropriate given they are both stablecoins. Do you agree?

**arjun-io**

> @arjun-io @panprog @bin2chen66 For the current supported tokens in READ.ME, I think medium severity remains appropriate given they are both stablecoins. Do you agree?

I'm not entirely sure how the stablecoin in use matters here? Returning an invalid versions as valid can be very detrimental in markets where invalid versions can be triggered at will (such as in markets that close) which can result in users being able to open or close positions when they shouldn't be able to

**sherlock-admin4**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/equilibria-xyz/perennial-v2/pull/308

---
### Example 2

**Auto Label:** Premature exits and flawed validation lead to missed callbacks, cascading failures, and unauthorized fee drainage, compromising oracle integrity and protocol reliability.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2023-10-perennial-judging/issues/26 

## Found by 
panprog
## Summary

According to protocol design (from KeeperOracle comments), multiple markets may use the same KeeperOracle instance:
```solidity
/// @dev One instance per price feed should be deployed. Multiple products may use the same
///      KeeperOracle instance if their payoff functions are based on the same underlying oracle.
///      This implementation only supports non-negative prices.
```

However, if `KeeperOracle` is used by several `Market` instances, and one of them makes a request and is then paused before the settlement, `KeeperOracle` will be temporarily bricked until `Market` is unpaused. This happens, because `KeeperOracle.commit` will revert in market callback, as `commit` iterates through all requested markets and calls `update` on all of them, and `update` reverts if the market is paused.

This means that pausing of just 1 market will basically stop trading in all the other markets which use the same `KeeperOracle`, disrupting protocol usage. When `KeeperOracle.commit` always reverts, it's also impossible to switch oracle provider from upstream `OracleFactory`, because provider switch still requires the latest version of previous oracle to be commited, and it will be impossible to commit it (both valid or invalid, requested or unrequested).

Additionally, the market's `update` can also revert for some other reasons, for example if maker exceeds the maker limit after invalid oracle as described in the other issue.

And for another problem (although a low severity, but caused in the same lines), if too many markets are authorized to call `KeeperOracle.request`, the markets callback gas usage might exceed block limit, making it impossible to call `commit` due to not enough gas. Currently there is no limit of the amount of Markets which can be added to callback queue.

## Vulnerability Detail

`KeeperOracle.commit` calls back `update` in all markets which called `request` in the oracle version:
```solidity
for (uint256 i; i < _globalCallbacks[version.timestamp].length(); i++)
    _settle(IMarket(_globalCallbacks[version.timestamp].at(i)), address(0));
...
function _settle(IMarket market, address account) private {
    market.update(account, UFixed6Lib.MAX, UFixed6Lib.MAX, UFixed6Lib.MAX, Fixed6Lib.ZERO, false);
}
```

If any `Market` is paused, its `update` function will revert (notice the `whenNotPaused` modifier):
```solidity
    function update(
        address account,
        UFixed6 newMaker,
        UFixed6 newLong,
        UFixed6 newShort,
        Fixed6 collateral,
        bool protect
    ) external nonReentrant whenNotPaused {
```

This means that if any `Market` is paused, all the other markets will be unable to continue trading since `commit` in their oracle provider will revert. It will also be impossible to successfully switch to a new provider for these markets, because previous oracle provider must still commit its latest request before fully switching to a new oracle provider:
```solidity
function _latestStale(OracleVersion memory currentOracleLatestVersion) private view returns (bool) {
    if (global.current == global.latest) return false;
    if (global.latest == 0) return true;

@@@ if (uint256(oracles[global.latest].timestamp) > oracles[global.latest].provider.latest().timestamp) return false;
    if (uint256(oracles[global.latest].timestamp) >= currentOracleLatestVersion.timestamp) return false;

    return true;
}
```

## Impact

One paused market will stop trading in all the markets which use the same oracle provider (`KeeperOracle`).

## Code Snippet

`KeeperOracle.commit` iterates all requested markets and settles them:
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L123-L124

`_settle` calls `update` on the market:
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L176-L178

`Market.update` has `whenNotPaused` modifier, making it revert when paused:
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L87

## Tool used

Manual Review

## Recommendation

1. Consider catching and ignoring revert, when calling `update` for the market in the `_settle` (wrap in try .. catch).
2. Consider adding a limit of the number of markets which are added to callback queue in each oracle version, or alternatively limit the number of authorized markets to call `request`.



## Discussion

**kbrizzle**

Markets are currently only pausable Factory-wide, which means this cannot happen unless there is a multi-MarketFactory setup pointing at the same Oracle instance.

While valid, we currently do not support this usage pattern, and this would be among many improvements we'd need to make to.

**panprog**

This issue is not limited to paused markets, but can happen for any reasons when market.update reverts, for example in the current codebase this can happen if maker exceeds makerLimit (issue #24), which will revert all update calls, subsequently bricking oracle update for all markets. This is mentioned in the issue description:
> Additionally, the market's `update` can also revert for some other reasons, for example if maker exceeds the maker limit after invalid oracle as described in the other issue.

I think this should be medium. It is still valid with paused markets (even if not considered supported setup by sponsor), but also valid if any other issue causes market to revert updates. This issue will make #24 more severe (#24 bricks 1 market, #26 makes it brick oracle commit and all markets using the same oracle)

**kbrizzle**

Thanks for the additional color.

We'd like to preserve the guarantee that each posted price will atomically settle the attached market(s) (globally / async for locally) to that version. This is important for a number of parameter improvements and future upgrades we have planned.

If there are settlement-revert cases that are possible given this paradigm, we'd like to address those as if they are market-bricking issues.

We're open to however you think this is fair to judge on a severity basis, but we will only be resolving actual revert issues versus making the settlement callback `try...catch`.

**panprog**

These are the planned future upgrades, but according to Sherlock rules it should be judged based on current code. In the current code there are no problems if market is not settled, but there are problems if due to some other issues (such as #24) the issue described here makes single market failure cause all the markets using the same oracle revert.

I'd like to add that while multi-factory setup is not supported, multi markets (from the same factory) pointing to the same oracle instance is supported. So the following setup is supported:

- MarketFactory1 deploys Market1 and Market2
- OracleFactory1 deploys Oracle1
- Market1 oracle is set to Oracle1 (say, it uses no payoff)
- Market2 oracle is set to Oracle1 (say, it uses 2x payoff - so a 2x market for the same underlying oracle)

In such setup, if Market1 is paused, then Market2 is paused too (because they're paused via MarketFactory1, markets don't have pause function by themselves). However, if Market1 maker exceeds makerLimit due to #24, then not only Market1 is bricked (reverts all updates), but also both Oracle1 and Market2 are bricked too (revert all commit and update transactions).

So while most of this issue description is about pause function (since I didn't know about multi-factory setup not being supported), which can be considered invalid due to not being supported, the description also does mention the other reasons for the issue to happen, including making the maker > makerLimit issue more severe (possibly some other issues which can revert update too). So this part is valid, so I believe it should be medium

---
### Example 3

**Auto Label:** Premature exits and flawed validation lead to missed callbacks, cascading failures, and unauthorized fee drainage, compromising oracle integrity and protocol reliability.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2023-10-perennial-judging/issues/25 

## Found by 
bin2chen, panprog, rvierdiiev
## Summary

The new feature introduced in 2.1 is the callback called for all markets and market+account pairs which requested the oracle version. These callbacks are called once the corresponding oracle settles. For this reason, `KeeperOracle` keeps a list of markets and market+account pairs per oracle version to call market.update on them:
```solidity
/// @dev Mapping from version to a set of registered markets for settlement callback
mapping(uint256 => EnumerableSet.AddressSet) private _globalCallbacks;

/// @dev Mapping from version and market to a set of registered accounts for settlement callback
mapping(uint256 => mapping(IMarket => EnumerableSet.AddressSet)) private _localCallbacks;
```

However, currently `KeeperOracle` stores only the market+account from the first request call per oracle version, because if the request was already made, it returns from the function before adding to the list:
```solidity
function request(IMarket market, address account) external onlyAuthorized {
    uint256 currentTimestamp = current();
@@@ if (versions[_global.currentIndex] == currentTimestamp) return;

    versions[++_global.currentIndex] = currentTimestamp;
    emit OracleProviderVersionRequested(currentTimestamp);

    // @audit only the first request per version reaches these lines to add market+account to callback list
    _globalCallbacks[currentTimestamp].add(address(market));
    _localCallbacks[currentTimestamp][market].add(account);
    emit CallbackRequested(SettlementCallback(market, account, currentTimestamp));
}
```

## Vulnerability Detail

According to docs, the same `KeeperOracle` can be used by multiple markets. And every account requesting in the same oracle version is supposed to be called back (settled) once the oracle version settles.

## Impact

The new core function of the protocol doesn't work as expected and `KeeperOracle` will fail to call back markets and accounts if there is more than 1 request in the same oracle version (which is very likely).

## Code Snippet

`KeeperOracle.request` will return early if the request for this oracle version was already made:
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L77

The lines to add market+account to callback list will only be reached once per oracle version:
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L82-L83

## Tool used

Manual Review

## Recommendation

Move addition to callback list to just before the condition to exit function early:
```solidity
function request(IMarket market, address account) external onlyAuthorized {
    uint256 currentTimestamp = current();
    _globalCallbacks[currentTimestamp].add(address(market));
    _localCallbacks[currentTimestamp][market].add(account);
    emit CallbackRequested(SettlementCallback(market, account, currentTimestamp));
    if (versions[_global.currentIndex] == currentTimestamp) return;

    versions[++_global.currentIndex] = currentTimestamp;
    emit OracleProviderVersionRequested(currentTimestamp);
}
```

---
