# Cluster -1124

**Rank:** #449  
**Count:** 7  

## Label
Misuse of `msg.sender` in external functions leads to incorrect state access, unintended defaults, and inconsistent behavior for external callers—compromising accuracy and reliability of returned values.

## Cluster Information
- **Total Findings:** 7

## Examples

### Example 1

**Auto Label:** Misuse of `msg.sender` in multicall contexts leads to incorrect sender identity, enabling unauthorized access, logic errors, and MEV exploitation due to failure to track original transaction originators.  

**Original Text Preview:**

**Description:** `LibMulticaller` is used throughout the codebase to retrieve the actual sender of multicall transactions; however, `BunniToken::_beforeTokenTransfer` and `BunniToken::_afterTokenTransfer` both incorrectly pass `msg.sender` directly to the corresponding Hooklet function:

```solidity
function _beforeTokenTransfer(address from, address to, uint256 amount, address newReferrer) internal override {
    IHooklet hooklet_ = hooklet();
    if (hooklet_.hasPermission(HookletLib.BEFORE_TRANSFER_FLAG)) {
        hooklet_.hookletBeforeTransfer(msg.sender, poolKey(), this, from, to, amount);
    }
}

function _afterTokenTransfer(address from, address to, uint256 amount) internal override {
    // call hooklet
    IHooklet hooklet_ = hooklet();
    if (hooklet_.hasPermission(HookletLib.AFTER_TRANSFER_FLAG)) {
        hooklet_.hookletAfterTransfer(msg.sender, poolKey(), this, from, to, amount);
    }
}
```

**Impact:** The Hooklet calls will reference the incorrect sender. This has potentially serious downstream effects for integrators as custom logic is executed with the incorrect address in multicall transactions.

**Recommended Mitigation:** `LibMulticaller.senderOrSigner()` should be used in place of `msg.sender` wherever the actual sender is required:

```diff
function _beforeTokenTransfer(address from, address to, uint256 amount, address newReferrer) internal override {
    IHooklet hooklet_ = hooklet();
    if (hooklet_.hasPermission(HookletLib.BEFORE_TRANSFER_FLAG)) {
--      hooklet_.hookletBeforeTransfer(msg.sender, poolKey(), this, from, to, amount);
++      hooklet_.hookletBeforeTransfer(LibMulticaller.senderOrSigner(), poolKey(), this, from, to, amount);
    }
}

function _afterTokenTransfer(address from, address to, uint256 amount) internal override {
    // call hooklet
    IHooklet hooklet_ = hooklet();
    if (hooklet_.hasPermission(HookletLib.AFTER_TRANSFER_FLAG)) {
--      hooklet_.hookletAfterTransfer(msg.sender, poolKey(), this, from, to, amount);
++      hooklet_.hookletAfterTransfer(LibMulticaller.senderOrSigner(), poolKey(), this, from, to, amount);
    }
}
```

**Bacon Labs:** Fixed in [PR \#106](https://github.com/timeless-fi/bunni-v2/pull/106).

**Cyfrin:** Verified, the `LibMulticaller` is now used to pass the `msg.sender` in `BunniToken` Hooklet calls.

---
### Example 2

**Auto Label:** Misuse of `msg.sender` in multicall contexts leads to incorrect sender identity, enabling unauthorized access, logic errors, and MEV exploitation due to failure to track original transaction originators.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Low

## Description

The `g8keepBondingCurve` contract utilizes the `SafeTransferLib.forceSafeTransferETH` function at line 840 to transfer Ether safely, as part of a reward mechanism. This function operates by attempting to send the value to the address, and if that fails, it creates a temporary contract holding the balance and uses self-destruction to forcibly send the ETH to the address, this is done to ensure the migration does not fail under any circumstances.

```solidity
File: g8keepBondingCurve.sol
840:         SafeTransferLib.forceSafeTransferETH(DEPLOYER, DEPLOYER_REWARD, 50_000);
```

However, there is a potential issue regarding the determination of the `DEPLOYER` address, which is set as `msg.sender` when the `deployToken` function is called in the `g8keepBondingCurveFactory` contract.

```solidity
File: g8keepBondingCurveFactory.sol
187:                     DEPLOYER: msg.sender,
```

The problem arises when the `deployToken` function is invoked by a `Multicaller` or any intermediate contract. In such cases, the intermediary contract becomes `msg.sender` and consequently the designated `DEPLOYER`, the ETH would be forcibly sent into the contract's balance. This results in the intermediary contract receiving the Ether transfer, rather than the intended user. This can lead to a race condition where malicious actors, such as MEV bots, exploit this situation to capture the Ether.

Additionally, when the token is migrated, the Uniswap position is sent to the **DEPLOYER**, which could potentially result in the loss of this position.

```solidity
File: g8keepBondingCurve.sol
837:         UNISWAP_POSITION_MANAGER.approve(address(LOCKER_FACTORY), lpTokenId);
838:         LOCKER_FACTORY.deploy(lpTokenId, DEPLOYER);
```

## Recommendations

Instead of using `msg.sender` as the **DEPLOYER** when calling `g8keepBondingCurveFactory::deployToken`, consider adding a parameter to specify the deployer reward recipient, ensuring it is eligible to receive the tokens.

---
### Example 3

**Auto Label:** Misuse of `msg.sender` in external functions leads to incorrect state access, unintended defaults, and inconsistent behavior for external callers—compromising accuracy and reliability of returned values.  

**Original Text Preview:**

### Relevant GitHub Links
<a data-meta="codehawks-github-link" href="https://github.com/Cyfrin/2024-05-beanstalk-the-finale/blob/8c8710df547f7d7c5dd82c5381eb6b34532e4484/protocol/contracts/beanstalk/farm/TractorFacet.sol#L130-L156">https://github.com/Cyfrin/2024-05-beanstalk-the-finale/blob/8c8710df547f7d7c5dd82c5381eb6b34532e4484/protocol/contracts/beanstalk/farm/TractorFacet.sol#L130-L156</a>


## Summary
TractorFacet will always return the wrong value for `getCounter` and `updateCounter` functions. But those can only work while running through a Blueprint call(while the activePublisher is set). This will prevent users to make external calls to the `TractorFacet` and consult their data related to the Tractor counter.

This issue occurs because the protocol is trying to fetch the `activePublisher` from `tractorStorage()` which has the address set to `1` and this address only changes when the `activePublisher` is set inside the `runBlueprint` modifier. 

Note that those functions are `external` and should return the correct value when the `msg.sender` matches the address of the publisher for that `counterId`. 

```solidity
    function getCounter(bytes32 counterId) public view returns (uint256 count) {
        return
            LibTractor._tractorStorage().blueprintCounters[
@>                LibTractor._tractorStorage().activePublisher // @audit current address is 1
            ][counterId];
    }


    /**
     * @notice Update counter value.
     * @return count New value of counter
     */
    function updateCounter(
        bytes32 counterId,
        LibTractor.CounterUpdateType updateType,
        uint256 amount
    ) external returns (uint256 count) {
        uint256 newCount;
        if (updateType == LibTractor.CounterUpdateType.INCREASE) {
            newCount = getCounter(counterId).add(amount);
        } else if (updateType == LibTractor.CounterUpdateType.DECREASE) {
            newCount = getCounter(counterId).sub(amount);
        }
        LibTractor._tractorStorage().blueprintCounters[
@>            LibTractor._tractorStorage().activePublisher // @audit current address is 1
        ][counterId] = newCount;
        return newCount;
    }
```



## Impact

- Users will not be able to update their Blueprint's counter as the `updateCounter` will set the `counterId` and its value to the `address(1)`
- Users will not be able to fetch their Blueprint's counter value as the `getCounter` will try to fetch the value from the `address(1)`

## PoC

Add the following functions to `TractorFacet` to facilitate the testing and then add `TractorFacet` to `BeanstalkDeployer`. 

```solidity
 // TractorFacet
 function setPublisher(address payable publisher) external {
        LibTractor._setPublisher(publisher);
    }
 function resetPublisher() external {
        LibTractor._resetPublisher();
    }
```

```diff
// BeanstalkDeployer
string[] facets = [
...
+ "TractorFacet" 
]
```

Now replace all the code on `Tractor.t.sol` with the following: 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

 import "forge-std/Test.sol";

 import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

 import {C} from "contracts/C.sol";
 import {LibAppStorage} from "contracts/libraries/LibAppStorage.sol";
 import {LibTractor} from "contracts/libraries/LibTractor.sol";
 import {IBeanstalk} from "contracts/interfaces/IBeanstalk.sol";
 import {TokenFacet} from "contracts/beanstalk/farm/TokenFacet.sol";
 import {TractorFacet} from "contracts/beanstalk/farm/TractorFacet.sol";
 import {TestHelper} from "test/foundry/utils/TestHelper.sol";
 import {LibTransfer} from "contracts/libraries/Token/LibTransfer.sol";
 import {LibClipboard} from "contracts/libraries/LibClipboard.sol";
 import {AdvancedFarmCall} from "contracts/libraries/LibFarm.sol";
 import {LibBytes} from "contracts/libraries/LibBytes.sol";


 contract TractorTest is TestHelper {
    IBeanstalk beanstalk;
    IERC20 bean;

    TractorFacet tractorFacet;
    TokenFacet tokenFacet;

    function setUp() public {
        initializeBeanstalkTestState(true, false);

        beanstalk = IBeanstalk(BEANSTALK);
        bean = IERC20(C.BEAN);
        tokenFacet = TokenFacet(BEANSTALK);
        tractorFacet = TractorFacet(BEANSTALK);
    }

    function testTractor_whenCallingGetCounter_willReturnWrongCounter() public {
        // pre conditions - set counterId. 
        bytes32 counterId = keccak256(abi.encodePacked("counterId"));
        _setPublisherWith(counterId);

        // action:  get the counter value - should return 500
        uint256 newValue = tractorFacet.getCounter(counterId);
        assertEq(newValue, 500);
    }

    function _setPublisherWith(bytes32 counterId) internal {
        tractorFacet.setPublisher(payable(address(this)));

        // set a new value for the counterId
        tractorFacet.updateCounter(counterId, LibTractor.CounterUpdateType.INCREASE, 500);
        uint256 value = tractorFacet.getCounter(counterId);
        assertEq(value, 500);

        // reset publisher - Beanstalk default state
        tractorFacet.resetPublisher();
    }
 }
```

Run: `forge test --match-test testTractor_whenCallingGetCounter_willReturnWrongCounter -vv` 

Output: 

```
Ran 1 test for test/foundry/tractor/Tractor.t.sol:TractorTest
[FAIL. Reason: assertion failed: 0 != 500] testTractor_whenCallingGetCounter_willReturnWrongCounter() (gas: 56703)
```

## Tools Used
Manual Review and Foundry

## Recommendations

Replace `LibTractor._tractorStorage().activePublisher` with `LibTractor._user()`. 

This works for both cases: active publisher set and `msg.sender`.

✅ Test with the fix:

```
[PASS] testTractor_whenCallingGetCounter_willReturnWrongCounter() (gas: 52451)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 248.29ms (444.13µs CPU time)
```

---
