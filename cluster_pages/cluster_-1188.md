# Cluster -1188

**Rank:** #380  
**Count:** 14  

## Label
Failure to validate transfer outcomes or state conditions leads to inconsistent or incorrect state updates, enabling fund loss, reentrancy, or misrepresentation of asset balances.

## Cluster Information
- **Total Findings:** 14

## Examples

### Example 1

**Auto Label:** Failure to validate transfer parameters leads to unintended state changes, unfair access, and inefficient resource usage, undermining protocol fairness and integrity.  

**Original Text Preview:**

**Description:** The purpose of `LINKMigrator` is that users can migrate their existing position from the Chainlink community pool to stake.link vaults, even when the community pool is at full utilization, since vacating a position frees up space. For users without an existing position, a queueing system (`PriorityQueue`) is used to wait for available slots in the community pool.

However, a user with an existing position can exploit this mechanism by faking a migration. By moving their position to another address (e.g., a small contract they control), they can bypass the queue and open a new position in stake.link if space is available.

Migration begins with a call to [`LINKMigrator::initiateMigration`](https://github.com/stakedotlink/contracts/blob/0bd5e1eecd866b2077d6887e922c4c5940a6b452/contracts/linkStaking/LINKMigrator.sol#L79-L92):

```solidity
function initiateMigration(uint256 _amount) external {
    if (_amount == 0) revert InvalidAmount();

    uint256 principal = communityPool.getStakerPrincipal(msg.sender);

    if (principal < _amount) revert InsufficientAmountStaked();
    if (!_isUnbonded(msg.sender)) revert TokensNotUnbonded();

    migrations[msg.sender] = Migration(
        uint128(principal),
        uint128(_amount),
        uint64(block.timestamp)
    );
}
```

Here, the user's principal is recorded. Later, the migration is completed via `transferAndCall`, which triggers [`LINKMigrator::onTokenTransfer`](https://github.com/stakedotlink/contracts/blob/0bd5e1eecd866b2077d6887e922c4c5940a6b452/contracts/linkStaking/LINKMigrator.sol#L100-L117):

```solidity
uint256 amountWithdrawn = migration.principalAmount -
    communityPool.getStakerPrincipal(_sender);
if (amountWithdrawn < _value) revert InsufficientTokensWithdrawn();
```

This compares the recorded and current principal to verify the withdrawal. However, it does not validate that the total staked amount in the community pool has decreased. As a result, a user can withdraw their position, transfer it to a contract they control, and still pass the check, allowing them to deposit directly into stake.link and bypass the queue.

**Impact:** A user with an existing position in the Chainlink community vault can circumvent the queue system and gain direct access to stake.link staking. This requires being in the claim period, having sufficient LINK to stake again, and available space in the Chainlink community vault. It also resets the bonding period, meaning the user would need to wait another 28 days (the Chainlink bonding period at the time of writing) before interacting with the new position. Nevertheless, this behavior could lead to unfair queue-skipping and undermine the fairness of the protocol.

**Proof of Concept:** Add the following test to `link-migrator.ts` which demonstrates the queue bypass by simulating a migration and re-staking via a third contract::
```javascript
it('can bypass queue using existing position', async () => {
  const { migrator, communityPool, accounts, token, stakingPool } = await loadFixture(
    deployFixture
  )

  // increase max pool size so we have space for the extra position
  await communityPool.setMaxPoolSize(toEther(3000))

  // deploy our small contract to hold the existing position
  const chainlinkPosition = (await deploy('ChainlinkPosition', [
    communityPool.target,
    token.target,
  ])) as ChainlinkPosition

  // get to claim period
  await communityPool.unbond()
  await time.increase(unbondingPeriod)

  // start batch transaction
  await ethers.provider.send('evm_setAutomine', [false])

  // 1. call initiate migration
  await migrator.initiateMigration(toEther(1000))

  // 2. unstake
  await communityPool.unstake(toEther(1000))

  // 3. transfer the existing position to a contract you control
  await token.transfer(chainlinkPosition.target, toEther(1000))
  await chainlinkPosition.deposit()

  // 4. transferAndCall a new position bypassing the queue
  await token.transferAndCall(
    migrator.target,
    toEther(1000),
    ethers.AbiCoder.defaultAbiCoder().encode(['bytes[]'], [[encodeVaults([])]])
  )
  await ethers.provider.send('evm_mine')
  await ethers.provider.send('evm_setAutomine', [true])

  // user has both a 1000 LINK position in stake.link StakingPool and chainlink community pool
  assert.equal(fromEther(await communityPool.getStakerPrincipal(accounts[0])), 0)
  assert.equal(fromEther(await stakingPool.balanceOf(accounts[0])), 1000)
  assert.equal(fromEther(await communityPool.getStakerPrincipal(chainlinkPosition.target)), 1000)

  // community pool is full again
  assert.equal(fromEther(await communityPool.getTotalPrincipal()), 3000)
  assert.equal(fromEther(await stakingPool.totalStaked()), 2000)
  assert.deepEqual(await migrator.migrations(accounts[0]), [0n, 0n, 0n])
})
```
Along with `ChainlinkPosition.sol`:
```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity 0.8.15;

import "./interfaces/IStaking.sol";
import "../core/interfaces/IERC677.sol";

contract ChainlinkPosition {

    IStaking communityPool;
    IERC677 link;

    constructor(address _communityPool, address _link) {
        communityPool = IStaking(_communityPool);
        link = IERC677(_link);
    }

    function deposit() public {
        link.transferAndCall(address(communityPool), link.balanceOf(address(this)), "");
    }
}
```

**Recommended Mitigation:** In `LINKMigrator::onTokenTransfer`, consider validating that the total principal in the community pool has decreased by at least `_value`, to ensure the migration reflects an actual exit from the community pool.

**stake.link:**
Fixed in [`de672a7`](https://github.com/stakedotlink/contracts/commit/de672a77813d507896502c20241618230af1bd85)

**Cyfrin:** Verified. Recommended mitigation was implemented. Community pool total principal is now recorded in `initiateMigration` then compared to the new pool total principal in `onTokenTransfer`.

\clearpage

---
### Example 2

**Auto Label:** Failure to validate transfer outcomes or state conditions leads to inconsistent or incorrect state updates, enabling fund loss, reentrancy, or misrepresentation of asset balances.  

**Original Text Preview:**

## Summary

The function provided by the `RToken` contract to [rescue any ERC20 tokens](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L337-L341) that were sent to the contract by mistake is [only callable by](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L54-L57) [`LendingPool`](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L54-L57).
However, `LendingPool` [does not offer any function](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/pools/LendingPool/LendingPool.sol#L731-L734) to call the `RToken` contract, resulting in unrecoverable funds.

## Vulnerability Details

The `Rtoken` contract provides [a function to rescue any ERC20 tokens](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L337-L341) that have been sent to the contract directly by mistake.
This function is only callable by the configured `reservePool`, which is ensured by the [`onlyReservePool`](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L54-L57) [modifier](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L54-L57).
This will be [`LendingPool`](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/pools/LendingPool/LendingPool.sol#L33) in practice.

Looking at the `LendingPool` contract, it comes indeed with a [`rescueToken`](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/pools/LendingPool/LendingPool.sol#L731-L734) [function](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/pools/LendingPool/LendingPool.sol#L731-L734), however, that one only allows for rescuing funds from itself.
It does not provide the necessary functionality to recover funds from the `RToken` contract.

Here's what it looks like:

```Solidity
function rescueToken(address tokenAddress, address recipient, uint256 amount) external onlyOwner {
    require(tokenAddress != reserve.reserveRTokenAddress, "Cannot rescue RToken");
    IERC20(tokenAddress).safeTransfer(recipient, amount);
}
```

Notice how it uses `safeTransfer` to move funds from itself to the `recipient`, but it does not call into the `RToken` contract.

## Impact

ERC20 tokens, that are not the configured reserve asset, which are sent to the `RToken` contract by mistake are not recoverable until the owner of the protocol configures a new `reservePool` that provides the necessary functionality

## Tools Used

Manual review.

## Recommendations

There's two ways to go about this:

1. Either allow the `owner` of the protocol to call `rescueFunds` on `RToken` directly or
2. Extend `LendingPool`, which is the expected `reservePool` in production, to provide the necessary function.

I'd recommend going for option 1) simply because any change of `reservePool` could reintroduce this issue.
Here's the necessary change:

```diff
- function rescueToken(address tokenAddress, address recipient, uint256 amount) external onlyReservePool {
+ function rescueToken(address tokenAddress, address recipient, uint256 amount) external onlyOwner {
    if (recipient == address(0)) revert InvalidAddress();
    if (tokenAddress == _assetAddress) revert CannotRescueMainAsset();
    IERC20(tokenAddress).safeTransfer(recipient, amount);
}
```

## Relevant links

* [`RToken#rescueToken`](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L337-L341)
* [`RToken#onlyReservePool`](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L54-L57)
* [`RToken#setReservePool`](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L85-L90)
* [`LendingPool#rescueToken`](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/pools/LendingPool/LendingPool.sol#L731-L734)

* [Similar finding other contest](https://solodit.cyfrin.io/issues/m-4-rescuetokens-feature-is-broken-sherlock-notional-leveraged-vaults-pendle-pt-and-vault-incentives-git)

---
### Example 3

**Auto Label:** Failure to validate critical preconditions before executing transfers or balance checks, leading to incorrect state transitions, fund locks, or misleading user feedback.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-03-axis-finance-judging/issues/21 

## Found by 
Aymen0909, KiroBrejka, ether\_sky, novaman33, sl1
## Summary
Seller's funds may remain locked in the protocol, because of revert on 0 transfer tokens.
In the README.md file is stated that the protocol uses every token with ERC20 Metadata and decimals between 6-18, which includes some revert on 0 transfer tokens, so this should be considered as valid issue!

## Vulnerability Detail
in the `AuctionHouse::claimProceeds()` function there is the following block of code:
```javascript
       uint96 prefundingRefund = routing.funding + payoutSent_ - sold_;
        unchecked {
            routing.funding -= prefundingRefund;
        }
        Transfer.transfer(
            routing.baseToken,
            _getAddressGivenCallbackBaseTokenFlag(routing.callbacks, routing.seller),
            prefundingRefund,
            false
        );
```
Since the batch auctions must be prefunded so `routing.funding` shouldn’t be zero unless all the tokens were sent in settle, in which case `payoutSent` will equal `sold_`. From this we make the conclusion that it is possible for `prefundingRefund` to be equal to 0. This means if the `routing.baseToken` is a revert on 0 transfer token the seller will never be able to get the `quoteToken` he should get from the auction.

## Impact
The seller's funds remain locked in the system and he will never be able to get them back.

## Code Snippet
The problematic block of code in the `AuctionHouse::claimProceeds()` function:
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/AuctionHouse.sol#L604-L613

`Transfer::transfer()` function, since it transfers the `baseToken`:
https://github.com/sherlock-audit/2024-03-axis-finance/blob/main/moonraker/src/lib/Transfer.sol#L49-L68

## Tool used

Manual Review

## Recommendation
Check if the `prefundingRefund > 0` like this:
```diff
   function claimProceeds(
        uint96 lotId_,
        bytes calldata callbackData_
    ) external override nonReentrant {
        // Validation
        _isLotValid(lotId_);

        // Call auction module to validate and update data
        (uint96 purchased_, uint96 sold_, uint96 payoutSent_) =
            _getModuleForId(lotId_).claimProceeds(lotId_);

        // Load data for the lot
        Routing storage routing = lotRouting[lotId_];

        // Calculate the referrer and protocol fees for the amount in
        // Fees are not allocated until the user claims their payout so that we don't have to iterate through them here
        // If a referrer is not set, that portion of the fee defaults to the protocol
        uint96 totalInLessFees;
        {
            (, uint96 toProtocol) = calculateQuoteFees(
                lotFees[lotId_].protocolFee, lotFees[lotId_].referrerFee, false, purchased_
            );
            unchecked {
                totalInLessFees = purchased_ - toProtocol;
            }
        }

        // Send payment in bulk to the address dictated by the callbacks address
        // If the callbacks contract is configured to receive quote tokens, send the quote tokens to the callbacks contract and call the onClaimProceeds callback
        // If not, send the quote tokens to the seller and call the onClaimProceeds callback
        _sendPayment(routing.seller, totalInLessFees, routing.quoteToken, routing.callbacks);

        // Refund any unused capacity and curator fees to the address dictated by the callbacks address
        // By this stage, a partial payout (if applicable) and curator fees have been paid, leaving only the payout amount (`totalOut`) remaining.
        uint96 prefundingRefund = routing.funding + payoutSent_ - sold_;
++ if(prefundingRefund > 0) { 
        unchecked {
            routing.funding -= prefundingRefund;
        }
            Transfer.transfer(
            routing.baseToken,
            _getAddressGivenCallbackBaseTokenFlag(routing.callbacks, routing.seller),
            prefundingRefund,
            false
        );
++        }
       

        // Call the onClaimProceeds callback
        Callbacks.onClaimProceeds(
            routing.callbacks, lotId_, totalInLessFees, prefundingRefund, callbackData_
        );
    }
```



## Discussion

**nevillehuang**

#21, #31 and #112 highlights the same issue of `prefundingRefund = 0`

#78 and #97  highlights the same less likely issue of `totalInLessFees = 0`

All points to same underlying root cause of such tokens not allowing transfer of zero, so duplicating them. Although this involves a specific type of ERC20, the impact could be significant given seller's fund would be locked permanently

**sherlock-admin4**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/Axis-Fi/moonraker/pull/142


**10xhash**

> The protocol team fixed this issue in the following PRs/commits: [Axis-Fi/moonraker#142](https://github.com/Axis-Fi/moonraker/pull/142)

Fixed
Now Transfer library only transfers token if amount > 0

**sherlock-admin4**

The Lead Senior Watson signed off on the fix.

---
