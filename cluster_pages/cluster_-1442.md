# Cluster -1442

**Rank:** #456  
**Count:** 7  

## Label
Failure to accurately track and include in-flight or transit assets in TVL calculations, leading to inflated or underreported TVL, flawed share pricing, and potential profit extraction or user fund loss.

## Cluster Information
- **Total Findings:** 7

## Examples

### Example 1

**Auto Label:** Inaccurate TVL reporting due to unaccounted token flows or missing fee/price calculations, enabling fund loss, front-running, and denial-of-service through flawed asset value tracking.  

**Original Text Preview:**

The protocol expects users to migrate their collateral from V1 vaults to V2 vaults, this significantly increases the TVL of the protocol's V2. At the same time, the Kerosene price depends on the TVL, in `UnboundedKerosineVault::assetPrice` the numerator of the equation is:

    uint256 numerator = tvl - dyad.totalSupply();

This will always revert until the TVL becomes `>` Dyad's supply, which is around 600k. So when users deposit Kerosene in either Kerosene vaults their Kerosene will temporarily get stuck in there.

### Proof of Concept

**This assumes that a reported bug is fixed, which is using the correct licenser. To overcome this, we had to manually change the licenser in `addKerosene` and `getKeroseneValue`.**

Because of another reported issue, a small change should be made to the code to workaround it, in `VaultManagerV2::withdraw`, replace `_vault.oracle().decimals()` with `8`. This just sets the oracle decimals to a static value of 8.

### Test POC:

Make sure to fork the main net and set the block number to `19703450`:

    contract VaultManagerTest is VaultManagerTestHelper {
        Kerosine keroseneV2;
        Licenser vaultLicenserV2;
        VaultManagerV2 vaultManagerV2;
        Vault ethVaultV2;
        VaultWstEth wstEthV2;
        KerosineManager kerosineManagerV2;
        UnboundedKerosineVault unboundedKerosineVaultV2;
        BoundedKerosineVault boundedKerosineVaultV2;
        KerosineDenominator kerosineDenominatorV2;
        OracleMock wethOracleV2;

        address bob = makeAddr("bob");
        address alice = makeAddr("alice");

        ERC20 wrappedETH = ERC20(MAINNET_WETH);
        ERC20 wrappedSTETH = ERC20(MAINNET_WSTETH);
        DNft dNFT = DNft(MAINNET_DNFT);

        function setUpV2() public {
            (Contracts memory contracts, OracleMock newWethOracle) = new DeployV2().runTestDeploy();

            keroseneV2 = contracts.kerosene;
            vaultLicenserV2 = contracts.vaultLicenser;
            vaultManagerV2 = contracts.vaultManager;
            ethVaultV2 = contracts.ethVault;
            wstEthV2 = contracts.wstEth;
            kerosineManagerV2 = contracts.kerosineManager;
            unboundedKerosineVaultV2 = contracts.unboundedKerosineVault;
            boundedKerosineVaultV2 = contracts.boundedKerosineVault;
            kerosineDenominatorV2 = contracts.kerosineDenominator;
            wethOracleV2 = newWethOracle;

            vm.startPrank(MAINNET_OWNER);
            Licenser(MAINNET_VAULT_MANAGER_LICENSER).add(address(vaultManagerV2));
            boundedKerosineVaultV2.setUnboundedKerosineVault(unboundedKerosineVaultV2);
            vm.stopPrank();
        }

        function test_InvalidCalculationAssetPrice() public {
            setUpV2();

            deal(MAINNET_WETH, bob, 100e18);

            vm.prank(MAINNET_OWNER);
            keroseneV2.transfer(bob, 100e18);

            uint256 bobNFT = dNFT.mintNft{value: 1 ether}(bob);

            vm.startPrank(bob);

            wrappedETH.approve(address(vaultManagerV2), type(uint256).max);
            keroseneV2.approve(address(vaultManagerV2), type(uint256).max);

            vaultManagerV2.add(bobNFT, address(ethVaultV2));
            vaultManagerV2.addKerosene(bobNFT, address(unboundedKerosineVaultV2));

            vaultManagerV2.deposit(bobNFT, address(ethVaultV2), 1e18);
            vaultManagerV2.deposit(bobNFT, address(unboundedKerosineVaultV2), 1e18);

            vm.roll(1);

            vm.expectRevert(); // Underflow
            vaultManagerV2.withdraw(bobNFT, address(unboundedKerosineVaultV2), 1e18, bob);
        }
    }

### Recommended Mitigation Steps

This is a bit tricky, but I think the most straightforward and logical solution would be to block the usage of the Kerosene vaults (just keep them unlicensed) until enough users migrate their positions from V1, i.e. the TVL reaches the Dyad's total supply.

### Assessed type

Under/Overflow

**[shafu0x (DYAD) confirmed and commented](https://github.com/code-423n4/2024-04-dyad-findings/issues/308#issuecomment-2089262068):**
 > Yes, it should only check for dyad minted from v1.

***

---
### Example 2

**Auto Label:** Inaccurate TVL reporting due to unaccounted token flows or missing fee/price calculations, enabling fund loss, front-running, and denial-of-service through flawed asset value tracking.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-02-perpetual-judging/issues/117 

The protocol has acknowledged this issue.

## Found by 
IllIllI
## Summary

Deposits/withdrawals of base tokens to the SpotHedgeBaseMaker aren't accounted for in the CircuitBreaker's accounting, so the tokens can be used by attackers to increase the amount withdrawable past the high water mark percentage limits.


## Vulnerability Detail

The SpotHedgeBaseMaker allows LPs to deposit base tokens in exchange for shares. The CircuitBreaker doesn't include base tokens in its accounting, until they're converted to quote tokens and added to the vault, which happens when someone opens a short base position, and the SpotHedgeBaseMaker needs to hedge its corresponding long base position, by swapping base tokens for the quote token. The CircuitBreaker keeps track of net deposits/withdrawals of the quote token using a high water mark system, in which the high water mark isn't updated until the sync interval has passed. As long as the _net_ outflow between sync intervals doesn't pass the threshold, withdrawals are allowed.


## Impact

Assume there is some sort of exploit, where the attacker is able to artificially inflate their 'fund' amount (e.g. one of my other submissions), and are looking to withdraw their ill-gotten gains. Normally, the amount they'd be able to withdraw would be limited to X% of the TVL of collateral tokens in the vault. By converting some of their 'fund' to margin, and opening huge short base positions against the SpotHedgeBaseMaker, they can cause the SpotHedgeBaseMaker to swap its contract balance of base tokens into collateral tokens, and deposit them into its vault account, increasing the TVL to TVL+Y. Since the time-based threshold will still be the old TVL, they're now able to withdraw `TVL.old * X% + Y`, rather than just `TVL.old * X%`. Depending on the price band limits set for swaps, the attacker can use a flash loan to temporarily skew the base/quote UniswapV3 exchange rate, such that the swap nets a much larger number of quote tokens than would normally be available to trade for. This means that if the 'fund' amount that the attacker has control over is larger than the actual number of collateral tokens in the vault, the attacker can potentially withdraw 100% of the TVL.


## Code Snippet

Quote tokens acquired via the swap are deposited into the [vault](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L571), which passes them through the [CircuitBreaker](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/vault/Vault.sol#L339):
```solidity
// File: src/maker/SpotHedgeBaseMaker.sol : SpotHedgeBaseMaker.fillOrder()   #1

415                } else {
416                    quoteTokenAcquired = _formatPerpToSpotQuoteDecimals(amount);
417 @>                 uint256 oppositeAmountXSpotDecimals = _uniswapV3ExactOutput(
418                        UniswapV3ExactOutputParams({
419                            tokenIn: address(_getSpotHedgeBaseMakerStorage().baseToken),
420                            tokenOut: address(_getSpotHedgeBaseMakerStorage().quoteToken),
421                            path: path,
422                            recipient: maker,
423                            amountOut: quoteTokenAcquired,
424                            amountInMaximum: _getSpotHedgeBaseMakerStorage().baseToken.balanceOf(maker)
425                        })
426                    );
427                    oppositeAmount = _formatSpotToPerpBaseDecimals(oppositeAmountXSpotDecimals);
428                    // Currently we don't utilize fillOrderCallback for B2Q swaps,
429                    // but we still populate the arguments anyways.
430                    fillOrderCallbackData.amountXSpotDecimals = quoteTokenAcquired;
431                    fillOrderCallbackData.oppositeAmountXSpotDecimals = oppositeAmountXSpotDecimals;
432                }
433    
434                // Deposit the acquired quote tokens to Vault.
435:@>             _deposit(vault, _marketId, quoteTokenAcquired);
```
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L405-L442

The CircuitBreaker only tracks [net liqInPeriod changes](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/circuitBreaker/LibLimiter.sol#L39-L49) changes during the withdrawal period, and only triggers the CircuitBreaker based on the TVL older than the withdrawal period:

```solidity
// File: src/circuitBreaker/LibLimiter.sol : LibLimiter.status()   #2

119            int256 currentLiq = limiter.liqTotal;
...
126 @>         int256 futureLiq = currentLiq + limiter.liqInPeriod;
127            // NOTE: uint256 to int256 conversion here is safe
128            int256 minLiq = (currentLiq * int256(limiter.minLiqRetainedBps)) / int256(BPS_DENOMINATOR);
129    
130 @>         return futureLiq < minLiq ? LimitStatus.Triggered : LimitStatus.Ok;
131:       }
```
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/circuitBreaker/LibLimiter.sol#L119-L131


## Tool used

Manual Review


## Recommendation

Calculate the quote-token value of each base token during LP `deposit()`/`withdraw()`, and add those amounts as CircuitBreaker flows during `deposit()`/`withdraw()`, then also invert those flows prior to depositing into/withdrawing from the vault



## Discussion

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**santipu_** commented:
> Medium/Low - Attacker would have to open a big short position against SpotHedgeBaseMaker during some time, exposing itself to losses due to price changes



**tailingchen**

Valid but won't fix.

Although circuit breaker can be bypassed, it still depends on the liquidity of SpotHedgeBaseMaker and the price band.
The team prefers not to fix it in the early stages. 

If the team want to fix it in the future, we have discussed several possible solutions. These solutions all have some pros and cons. The team is still evaluating possible options.
1. Separately calculate inflows and outflows for the same period. Only previous TVL is taken.
2. Do not include whitelisted maker’s deposit into TVL calculations.
3. Let SHBM check the current rate limit status of CircuitBreaker before depositing collateral into the vault. If the remaining balance that can be withdrawn is too small, it means that the current vault risk is too high and there is a risk that it cannot be withdrawn. Therefore, SHBM can refuse this position opening.

---
### Example 3

**Auto Label:** Inaccurate TVL reporting due to unaccounted token flows or missing fee/price calculations, enabling fund loss, front-running, and denial-of-service through flawed asset value tracking.  

**Original Text Preview:**

Users can lose a portion of their deposited funds if some of their funds haven't been deposited to the underlying Uniswap pools. There's always a chance of such event since Uniswap pools take balanced token amounts when liquidity is added but `GeVault` doesn't pre-compute balanced amounts. As a result, depositing and withdrawing can result in a partial loss of funds.

### Proof of Concept

The [GeVault.deposit()](https://github.com/code-423n4/2023-08-goodentry/blob/4b785d455fff04629d8675f21ef1d1632749b252/contracts/GeVault.sol#L247) function is used by users to deposits funds into ticks and underlying Uniswap pools. The function takes funds from the caller and calls `rebalance()` to distribute the funds among the ticks. The [GeVault.rebalance()](https://github.com/code-423n4/2023-08-goodentry/blob/4b785d455fff04629d8675f21ef1d1632749b252/contracts/GeVault.sol#L202) function first removes liquidity from all ticks and then deposits the removed assets plus the user assets back in to the ticks:

```solidity
function rebalance() public {
    require(poolMatchesOracle(), "GEV: Oracle Error");
    removeFromAllTicks();
    if (isEnabled) deployAssets();
}
```

The `GeVault.deployAssets()` function calls the [GeVault.depositAndStash()](https://github.com/code-423n4/2023-08-goodentry/blob/4b785d455fff04629d8675f21ef1d1632749b252/contracts/GeVault.sol#L404) function, which actually deposits tokens into a `TokenisableRange` contract by calling the [TokenisableRange.deposit()](https://github.com/code-423n4/2023-08-goodentry/blob/4b785d455fff04629d8675f21ef1d1632749b252/contracts/TokenisableRange.sol#L222). The function deposits tokens into a Uniswap V3 pool and returns unspent tokens to the caller:

```solidity
(uint128 newLiquidity, uint256 added0, uint256 added1) = POS_MGR.increaseLiquidity(
    ...
);

...

_mint(msg.sender, lpAmt);
TOKEN0.token.safeTransfer( msg.sender, n0 - added0);
TOKEN1.token.safeTransfer( msg.sender, n1 - added1);
```

However, the `GeVault.depositAndStash()` function doesn't handle the returned unspent tokens. Since Uniswap V3 pools take balanced token amounts (respective to the current pool price) and since the funds deposited into ticks are not balanced ([`deployAssets()` splits token amounts in halves](https://github.com/code-423n4/2023-08-goodentry/blob/4b785d455fff04629d8675f21ef1d1632749b252/contracts/GeVault.sol#L353-L358)), there's always a chance that the `TokenisableRange.deposit()` function won't consume all specified tokens and will return some of them to the `GeVault` contract. However, `GeVault` won't return the unused tokens to the depositor.

Moreover, the contract won't include them in the TVL calculation:

1.  The [GeVault.getTVL()](https://github.com/code-423n4/2023-08-goodentry/blob/4b785d455fff04629d8675f21ef1d1632749b252/contracts/GeVault.sol#L392) function computes the total LP token balance of the contract (`getTickBalance(k)`) and the price of each LP token (`t.latestAnswer()`), to compute the total value of the vault.
2.  The [GeVault.getTickBalance()](https://github.com/code-423n4/2023-08-goodentry/blob/4b785d455fff04629d8675f21ef1d1632749b252/contracts/GeVault.sol#L420) function won't count the unused tokens because it only returns the amount of LP tokens deposited into the lending pool. I.e. only the liquidity deposited to Uniswap pools is counted.
3.  The [TokenisableRange.latestAnswer()](https://github.com/code-423n4/2023-08-goodentry/blob/4b785d455fff04629d8675f21ef1d1632749b252/contracts/TokenisableRange.sol#L361) function computes the total value ([TokenisableRange.sol#L355](https://github.com/code-423n4/2023-08-goodentry/blob/4b785d455fff04629d8675f21ef1d1632749b252/contracts/TokenisableRange.sol#L355)) of the liquidity deposited into the Uniswap pool ([TokenisableRange.sol#L338](https://github.com/code-423n4/2023-08-goodentry/blob/4b785d455fff04629d8675f21ef1d1632749b252/contracts/TokenisableRange.sol#L338)). Thus, the unused tokens won't be counted here as well.
4.  The [GeVault.getTVL()](https://github.com/code-423n4/2023-08-goodentry/blob/4b785d455fff04629d8675f21ef1d1632749b252/contracts/GeVault.sol#L220-L223) function is used to compute the amount of tokens to return to the depositor during withdrawal.

Thus, the unused tokens will be locked in the contract until they're deposited into ticks. However, rebalancing and depositing of tokens can result in new unused tokens that won't be counted in the TVL.

### Recommended Mitigation Steps

In the `GeVault.deposit()` function, consider returning unspent tokens to the depositor. Extra testing is needed to guarantee that rebalancing doesn't result in unspent tokens, or, alternatively, such tokens could be counted in a storage variable and excluded from the balance of unspent tokens during depositing.

Alternatively, consider counting `GeVault`'s token balances in the `getTVL()` function. This won't require returning unspent tokens during depositing and will allow depositors to withdraw their entire funds.

**[Keref (Good Entry) confirmed and commented](https://github.com/code-423n4/2023-08-goodentry-findings/issues/325#issuecomment-1683244287):**
 > See [update](https://github.com/GoodEntry-io/ge/commit/a8ba6492b19154c72596086f5531f6821b4a46a2).

**[Good Entry Mitigated](https://github.com/code-423n4/2023-09-goodentry-mitigation#individual-prs):**
> Take unused funds into account for TVL.<br>
> PR: https://github.com/GoodEntry-io/ge/commit/a8ba6492b19154c72596086f5531f6821b4a46a2

**Status:** Mitigation confirmed. Full details in reports from  [kutugu](https://github.com/code-423n4/2023-09-goodentry-mitigation-findings/issues/34), [xuwinnie](https://github.com/code-423n4/2023-09-goodentry-mitigation-findings/issues/28) and [3docSec](https://github.com/code-423n4/2023-09-goodentry-mitigation-findings/issues/6).

***

---
