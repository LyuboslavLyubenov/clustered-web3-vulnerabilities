# Cluster -1174

**Rank:** #171  
**Count:** 73  

## Label
Failure to validate success or state conditions in critical operations leads to irreversible fund loss, denial of service, or unauthorized fund drainage due to inadequate error handling and lack of runtime enforcement.

## Cluster Information
- **Total Findings:** 73

## Examples

### Example 1

**Auto Label:** Improper handling of external calls leads to unhandled reverts, incorrect state, or arbitrary execution—enabling unexpected reverts, corrupted data, or unauthorized actions in state-changing or read-only operations.  

**Original Text Preview:**

##### Description

In the `UmbrellaBatchHelper` contract, there is a potential issue with error handling in the following try-catch block:

```solidity
try IUniversalToken(underlyingOfStakeToken).aToken() returns (address _aToken)
```
https://github.com/bgd-labs/aave-umbrella-private/blob/e3dde13fece757561fc199a1523470b6764f8930/src/contracts/helpers/UmbrellaBatchHelper.sol#L324

The `aToken()` call is made using `staticcall` since it's a view function. If the `underlyingOfStakeToken` contract has a `fallback()` function **that doesn't modify state**, a decoding error could occur that won't be caught by the catch clause and the whole transaction will revert.

While the code works correctly with tokens like WETH (where the fallback function modifies state and the `staticcall` revert is caught), there's a possibility of encountering tokens with fallback functions that don't modify state. In such cases, the decoding error would not be caught, leading to unexpected behavior.

##### Recommendation

We recommend considering using `staticcall` directly and checking its return value.

***

---
### Example 2

**Auto Label:** Failure to validate contract existence or call success leads to silent, erroneous executions and permanent data loss.  

**Original Text Preview:**

**Description:** `TimelockController::_execute` does this:
```solidity
function _execute(address target, uint256 value, bytes calldata data) internal virtual {
    (bool success, bytes memory returndata) = target.call{value: value}(data);
    Address.verifyCallResult(success, returndata);
}
```

If `target` is a non-existent contract but `data` contains a valid expected function call with parameters, the `call` will return `true`; `Address.verifyCallResult` fails to catch this case.

**Proof Of Concept:**
Add PoC function to `test/EthenaTimelockController.sol`:
```solidity
function testExecuteNonExistentContract() public {
    bytes memory data = abi.encodeWithSignature("DONTEXIST()");
        _scheduleWaitExecute(address(0x1234), data);
}
```

Run with: `forge test --match-test testExecuteNonExistentContract -vvv`

**Recommended Mitigation:** We reported this bug to OpenZeppelin but they said they prefer the current implementation as it is more flexible. We disagree with this assessment and believe it is incorrect for `TimelockController::_execute` to not revert when there is valid calldata but the target has no code.

**Ethena:** Fixed in commit [e58c547](https://github.com/ethena-labs/timelock-contract/commit/e58c547e3bcbea79d9df7121b5bb04626a2b72e0#diff-8ca72e61ebf9a693737b5c9052aa3814e8b291e3d6dd0341fe88b5b5e781427bR191-R197) by overriding `_execute` to revert if `data.length > 0 && target.code.length == 0`.

**Cyfrin:** Verified.

---
### Example 3

**Auto Label:** Failure to validate ERC20 transfer return values leads to silent failures, incorrect state, and potential loss of funds or denial-of-service due to improper error handling.  

**Original Text Preview:**

There are multiple occassions in the `staking.sol` contract where the `IERC20::transferFrom` and `IERC20::transfer` functions are used to transfer the `tokens` between `msg.sender` and contract itself.

But the issue is if the `token` being transferred does not return `bool` the above transfer transaction will fail since the `IERC20` implementation expects a `bool` value as the return value.

Since the `coin flip` game expects to work with multiple tokens whitelisted by the owner, this could be an issue.

Hence this will break the core functionalities of the game when such tokens are used in the game.

Hence it is recommended to use the `safeTransfer` and `safeTransferFrom` from the openzeppelin `SafeERC20` library.

---
