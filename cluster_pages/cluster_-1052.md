# Cluster -1052

**Rank:** #274  
**Count:** 30  

## Label
Redundant function calls and poor abstraction lead to inconsistent state, duplicated logic, and unpredictable behavior, increasing attack surface and undermining secure, deterministic token address management.

## Cluster Information
- **Total Findings:** 30

## Examples

### Example 1

**Auto Label:** Redundant function calls and poor abstraction lead to inconsistent state, duplicated logic, and unpredictable behavior, increasing attack surface and undermining secure, deterministic token address management.  

**Original Text Preview:**

##### Description

The `_predictStakeTokenAddress()` function calculates the StakeToken address using parameters such as `UMBRELLA_STAKE_TOKEN_IMPL()`, `SUPER_ADMIN()`, and `creationData`, which includes:
- `stakeSetup.underlying`,
- `name`,
- `symbol`,
- `stakeSetup.cooldown`,
- `stakeSetup.unstakeWindow`.

https://github.com/bgd-labs/aave-umbrella-private/blob/946a220a57b4ae0ad11d088335f9bcbb0e34dcef/src/contracts/umbrella/UmbrellaStkManager.sol#L239-L244

There are a number of inconsistencies in contract management and address prediction:

1. The `UmbrellaStkManager` contract stores the `umbrellaStakeTokenImpl` address at initialization and uses it when deploying new proxies for `UmbrellaStakeToken`. However, if the `UmbrellaStakeToken` implementation is upgraded, `UmbrellaStkManager.createStakeTokens()` will still create proxies with the outdated implementation, leading to inconsistencies between old and new stake tokens.

It is possible to upgrade the `Umbrella` contract to use a new `UmbrellaStakeToken` implementation, but it would make the function `_predictStakeTokenAddress()` to return incorrect addresses for older implementations.

2. Multiple tokens with the same `name`, `symbol` and `underlying` could be created using different `cooldown` and `unstakeWindow`.

3. In the created **StakeToken** contract, it is possible to update the **cooldown** and **unstakeWindow** parameters. This leads to uncertainty — when specifying these parameters along with their new values, the prediction function will return an address different from the one where the **StakeToken** was originally created, since the parameters have changed.

Moreover, considering that we can create two identical **StakeTokens** with the same symbol and underlying asset, and then modify the **cooldown** and **unstakeWindow** parameters in one of them to match the other, the uncertainty increases even further. This raises the question of what the prediction function should return in such cases.

##### Recommendation

If the function is intended to be used only during the deployment of new **StakeTokens**—to set the **StakeToken** address in other contracts before the actual deployment—then we recommend adding a warning in the comments stating that this function should not be used for any other purpose.

***

---
### Example 2

**Auto Label:** Inadequate interface abstraction and redundant reimplementation lead to unchecked state mutations, front-running risks, and loss of transparency in token balance management.  

**Original Text Preview:**

## OFTVotes Contract Simplification

## Context
**File:** OFTVotes.sol#L13

## Description
The OFTVotes contract can be simplified by inheriting from `OFTUpgradeable` instead of `OFT-CoreUpgradeable`. Currently, it defines four functions relevant to cross-chain transfers:

- `token()`
- `approvalRequired()`
- `_debit()`
- `_credit()`

All of these functions are the same as those defined in the `OFTUpgradeable` contract (see `OFTUpgradeable.sol`). Inheriting from `OFTUpgradeable` directly would simplify the codebase.

## Recommendation
For the OFTVotes contract, consider inheriting from `OFTUpgradeable` instead of `OFT-CoreUpgradeable`. If there is a need for custom implementation of any of the `OFTUpgradeable` functions, consider overriding them in the OFTVotes contract.

If the OFTVotes contract remains the same, consider removing the `_localDecimals` parameter in the constructor and changing `OFTCoreUpgradeable(_localDecimals, _endpoint)` to `OFTVotes(decimals(), _lzEndpoint)`. Since `_localDecimals` should represent the decimals of the OFT (18 decimals by default), this ensures no mismatch between `_localDecimals` and `decimals()`.

## Status
- **Bitcorn:** Fixed in PR 35.
- **Cantina Managed:** Verified.

---
### Example 3

**Auto Label:** Inadequate interface abstraction and redundant reimplementation lead to unchecked state mutations, front-running risks, and loss of transparency in token balance management.  

**Original Text Preview:**

## Context: WERC721.sol#L80-L81

## Description
The WERC721.sol contract acts as an extended ERC20.sol contract. The basic functionalities have been implemented without using the current standard from OpenZeppelin. Although there has not been any security identified issue, the OpenZeppelin implementation performs important sanity checks and controlled errors in a more accurate way.

## Recommendation
Consider inheriting from the ERC20.sol contract from OpenZeppelin (OZ).

## Comments
- **Sweep n' Flip:** I agree that OZ is a solid choice. Will consider.
- **Cantina Managed:** Acknowledged.

---
