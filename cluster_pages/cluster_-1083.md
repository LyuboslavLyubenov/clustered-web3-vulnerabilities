# Cluster -1083

**Rank:** #299  
**Count:** 24  

## Label
Redundant computations and unnecessary object operations lead to performance inefficiencies, increased gas costs, and resource exhaustion due to poor caching and misuse of lifecycle management.

## Cluster Information
- **Total Findings:** 24

## Examples

### Example 1

**Auto Label:** Redundant conditions and operations leading to unnecessary computations, increased gas costs, and degraded code efficiency due to poor logic design and lack of optimization.  

**Original Text Preview:**

##### Description
The issue has been identified within the [HanjiLOBFactory](https://github.com/longgammalabs/hanji-contracts/blob/70b15ec4d9e7578248141604503843716a67d875/src/HanjiLOBFactory.sol#L101) and [HanjiTrieFactory](https://github.com/longgammalabs/hanji-contracts/blob/70b15ec4d9e7578248141604503843716a67d875/src/HanjiTrieFactory.sol#L46) contracts.

These contracts exhibit inefficiencies and redundant operations in the deployment process. In the `HanjiLOBFactory` contract, deploying the `HanjiLOBProxy` involves unnecessary steps, such as toggling the `TrieFactory` status on and off by calling the `TrieFactory.setStatus` function. The status that controls whether the `TrieFactory` can create the trie is redundant, as this ability does not impact the security of the system. The `createTrie` function can be safely marked as callable by anyone, making the status control unnecessary.

The issue is classified as **low** severity because it does not directly threaten the security of the contracts but can lead to inefficiencies and potential errors in the deployment logic.
##### Recommendation
We recommend refactoring the deployment logic to eliminate the need for toggling the `TrieFactory` status. Specifically:
1. Remove the redundant `setStatus` function from the `HanjiTrieFactory` contract.
2. Streamline the `createHanjiLOB` function to deploy the `HanjiLOBProxy` more effectively without requiring intermediate status changes in the `TrieFactory`.

---
### Example 2

**Auto Label:** Redundant conditions and operations leading to unnecessary computations, increased gas costs, and degraded code efficiency due to poor logic design and lack of optimization.  

**Original Text Preview:**

##### Description
Several redundant conditions and operations were spotted across various functions:
- In the [`executeRight`](https://github.com/longgammalabs/hanji-contracts/blob/09b6188e028650b9c1758010846080c5f8c80f8e/src/TrieLib.sol#L216) function, condition `order_id > node_id` is always false and can be removed.
- In the [`calcCommonParent`](https://github.com/longgammalabs/hanji-contracts/blob/09b6188e028650b9c1758010846080c5f8c80f8e/src/TrieLib.sol#L836) function, the bitwise OR operation `|` is redundant when casting to `uint64`.
- In the [`normalizeBranch`](https://github.com/longgammalabs/hanji-contracts/blob/09b6188e028650b9c1758010846080c5f8c80f8e/src/TrieLib.sol#L335) function, the `new_rightmost_map` is always assigned to `rightmost_map` in the usages ([\[1\]](https://github.com/longgammalabs/hanji-contracts/blob/09b6188e028650b9c1758010846080c5f8c80f8e/src/TrieLib.sol#L455), [\[2\]](https://github.com/longgammalabs/hanji-contracts/blob/09b6188e028650b9c1758010846080c5f8c80f8e/src/TrieLib.sol#L529)) of this function. Updating `rightmost_map` directly without returning `new_rightmost_map` is recommended.
- In the [`executeRight`](https://github.com/longgammalabs/hanji-contracts/blob/09b6188e028650b9c1758010846080c5f8c80f8e/src/TrieLib.sol#L269), [`removeBestOffer`](https://github.com/longgammalabs/hanji-contracts/blob/09b6188e028650b9c1758010846080c5f8c80f8e/src/TrieLib.sol#L684) and [`removeOrderFromNodes`](https://github.com/longgammalabs/hanji-contracts/blob/09b6188e028650b9c1758010846080c5f8c80f8e/src/TrieLib.sol#L734) functions, the operations can be simplified by using `nodes[node_id].right = new_right_child` instead of loading the node first.
- The [`getAndUpdateNonce`](https://github.com/longgammalabs/hanji-contracts/blob/09b6188e028650b9c1758010846080c5f8c80f8e/src/OnchainLOB.sol#L601) function could be optimized to return the nonce before subtraction.
- In the [`placeOrder`](https://github.com/longgammalabs/hanji-contracts/blob/09b6188e028650b9c1758010846080c5f8c80f8e/src/OnchainLOB.sol#L316) function the `return order_id` statement is redudant, as it is already implicitly marked to return.
- In the [`getTraderByAddress`](https://github.com/longgammalabs/hanji-contracts/blob/09b6188e028650b9c1758010846080c5f8c80f8e/src/OnchainLOB.sol#L511) function, setting the new trader_id can be simplified to `trader.trader_id = last_used_trader_id++`
- In the [`receiveTokens`](https://github.com/longgammalabs/hanji-contracts/blob/09b6188e028650b9c1758010846080c5f8c80f8e/src/OnchainLOB.sol#L691-L692) function, the `trader` parameter is passed by memory, which is unnecessary as it can be modified directly by reference. Additionally, the `trader_changed` argument is redundant and not needed.
- The calls to `SafeERC20.safeTransfer|From` can be made more readable by using syntax `using SafeERC20 for IERC20;`. This allows the call to be written as `token_x.safeTransfer|From(...)`,
- The while loop in the [`calcCommonParent`](https://github.com/longgammalabs/hanji-contracts/blob/09b6188e028650b9c1758010846080c5f8c80f8e/src/TrieLib.sol#L833) function uses an inefficient bitwise operation to find the highest bit set. Utlizing OpenZeppelin's `Math.log2` function can perform this operation more efficiently and save gas.
- The usages of the [`shouldInsertToLeftChild`](https://github.com/longgammalabs/hanji-contracts/blob/09b6188e028650b9c1758010846080c5f8c80f8e/src/TrieLib.sol#L857) function are equivalent to condition `order_id < node_id`, making the custom function call redundant.
##### Recommendation
We recommend optimizing the operations enlisted above to improve the readability and reduce gas costs.

---
### Example 3

**Auto Label:** Redundant conditions and operations leading to unnecessary computations, increased gas costs, and degraded code efficiency due to poor logic design and lack of optimization.  

**Original Text Preview:**

##### Description
The issue has been identified within the [HanjiLOBFactory](https://github.com/longgammalabs/hanji-contracts/blob/70b15ec4d9e7578248141604503843716a67d875/src/HanjiLOBFactory.sol#L101) and [HanjiTrieFactory](https://github.com/longgammalabs/hanji-contracts/blob/70b15ec4d9e7578248141604503843716a67d875/src/HanjiTrieFactory.sol#L46) contracts.

These contracts exhibit inefficiencies and redundant operations in the deployment process. In the `HanjiLOBFactory` contract, deploying the `HanjiLOBProxy` involves unnecessary steps, such as toggling the `TrieFactory` status on and off by calling the `TrieFactory.setStatus` function. The status that controls whether the `TrieFactory` can create the trie is redundant, as this ability does not impact the security of the system. The `createTrie` function can be safely marked as callable by anyone, making the status control unnecessary.

The issue is classified as **low** severity because it does not directly threaten the security of the contracts but can lead to inefficiencies and potential errors in the deployment logic.
##### Recommendation
We recommend refactoring the deployment logic to eliminate the need for toggling the `TrieFactory` status. Specifically:
1. Remove the redundant `setStatus` function from the `HanjiTrieFactory` contract.
2. Streamline the `createHanjiLOB` function to deploy the `HanjiLOBProxy` more effectively without requiring intermediate status changes in the `TrieFactory`.

---
