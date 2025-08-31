# Cluster -1084

**Rank:** #34  
**Count:** 583  

## Label
Failure to validate or check contract state before execution leads to incorrect state updates, unauthorized operations, or silent failures, risking asset loss, inconsistent accounting, or exploitation during invalid or transitional states.

## Cluster Information
- **Total Findings:** 583

## Examples

### Example 1

**Auto Label:** Failure to validate critical inputs (addresses, chain IDs, token data) during initialization or execution, leading to invalid state, reentrancy, or unauthorized operations.  

**Original Text Preview:**

## Severity

**Impact**: High, malicious attacker can set L2 custom address to different address to break the bridge token.

**Likelihood**: Medium, attacker can front-ran the `registerTokenOnL2` to break the bridge token.

## Description

tSQD is designed so that it can be bridged from Ethereum (L1) to Arbitrum (L2) via Arbitrum’s generic-custom gateway.
However, the `registerTokenOnL2` function, which sets the L2 token address via `gateway.registerTokenToL2`, is not currently restricted.

```solidity
  function registerTokenOnL2(
    address l2CustomTokenAddress,
    uint256 maxSubmissionCostForCustomGateway,
    uint256 maxSubmissionCostForRouter,
    uint256 maxGasForCustomGateway,
    uint256 maxGasForRouter,
    uint256 gasPriceBid,
    uint256 valueForGateway,
    uint256 valueForRouter,
    address creditBackAddress
  ) public payable {
    require(!shouldRegisterGateway, "ALREADY_REGISTERED");
    shouldRegisterGateway = true;

    gateway.registerTokenToL2{value: valueForGateway}(
      l2CustomTokenAddress, maxGasForCustomGateway, gasPriceBid, maxSubmissionCostForCustomGateway, creditBackAddress
    );

    router.setGateway{value: valueForRouter}(
      address(gateway), maxGasForRouter, gasPriceBid, maxSubmissionCostForRouter, creditBackAddress
    );

    shouldRegisterGateway = false;
  }
```

An attacker can front-run the `registerTokenOnL2` and put an incorrect address for `l2CustomTokenAddress` to break the bridge token. Once it is called, the L2 token cannot be changed inside the gateway.

## Recommendations

Use the Ownable functionality inside tSQD and restrict `registerTokenOnL2` so that it can only be called by the owner/admin, as suggested by the Arbitrum bridge token design.

---
### Example 2

**Auto Label:** Insufficient validation of critical state conditions enables unauthorized manipulation of stake, rewards, or access, leading to fund loss, incorrect balances, or denial of service.  

**Original Text Preview:**

## Severity

**Impact:** High, user's funds will be stolen

**Likelihood:** High, can be exploited by anyone and easy to implement

## Description

`GatewayRegistry` contract allows users to register and stake tokens into gateways to receive computation units CUs. First, the user registers a gateway,

```solidity
  function register(bytes calldata peerId, string memory metadata, address gatewayAddress) public whenNotPaused {
    require(peerId.length > 0, "Cannot set empty peerId");
    bytes32 peerIdHash = keccak256(peerId);
    require(gateways[peerIdHash].operator == address(0), "PeerId already registered");

    gateways[peerIdHash] = Gateway({
      operator: msg.sender,
      peerId: peerId,
      strategy: defaultStrategy,
      ownAddress: gatewayAddress,
      metadata: metadata,
      totalStaked: 0,
>>    totalUnstaked: 0
    });
```

note `totalUnstaked` is set to 0. After this we can stake tokens

```solidity
  function _stakeWithoutTransfer(bytes calldata peerId, uint256 amount, uint128 durationBlocks) internal {
    (Gateway storage gateway, bytes32 peerIdHash) = _getGateway(peerId);
    _requireOperator(gateway);

    uint256 _computationUnits = computationUnitsAmount(amount, durationBlocks);
    uint128 lockStart = router.networkController().nextEpoch();
    uint128 lockEnd = lockStart + durationBlocks;
>>  stakes[peerIdHash].push(Stake(amount, _computationUnits, lockStart, lockEnd));
```

`stakes` mapping is used to track all user stakes. The problem arises when we unregister the gateway, we do not delete the `stakes`, it can be exploited in the following steps:

- unstake tokens from the gateway
- unregister gateway
- register new gateway with the same `peerId`
- since `totalUnstaked = 0`, we can unstake tokens again

```solidity
  function _unstakeable(Gateway storage gateway) internal view returns (uint256) {
    Stake[] memory _stakes = stakes[keccak256(gateway.peerId)];
    uint256 blockNumber = block.number;
    uint256 total = 0;
    for (uint256 i = 0; i < _stakes.length; i++) {
      Stake memory _stake = _stakes[i];
      if (_stake.lockEnd <= blockNumber) {
        total += _stake.amount;
      }
    }
    return total - gateway.totalUnstaked;
  }
```

Here is the coded POC for `GatewayRegistry.unstake.t.sol `

```solidity
  function test_StealStakes() public {
    uint256 amount = 100;
    address alice = address(0xA11cE);
    token.transfer(alice, amount);

    // stakers stake into their gateways
    gatewayRegistry.stake(peerId, amount, 200);
    vm.startPrank(alice);
    token.approve(address(gatewayRegistry), type(uint256).max);
    gatewayRegistry.register(bytes("alice"), "", address(0x6a7e));
    gatewayRegistry.stake(bytes("alice"), amount, 200);

    assertEq(token.balanceOf(address(gatewayRegistry)), 200);
    // exploit
    vm.roll(block.number + 300);
    gatewayRegistry.unstake(bytes("alice"), amount);
    gatewayRegistry.unregister(bytes("alice"));
    gatewayRegistry.register(bytes("alice"), "", address(0x6a7e));
    // unstake again
    gatewayRegistry.unstake(bytes("alice"), amount);
    assertEq(token.balanceOf(address(gatewayRegistry)), 0);
  }
```

## Recommendations

Delete `stakes` mapping when gateway is being unregistered.

---
### Example 3

**Auto Label:** Insufficient validation of critical state conditions enables unauthorized manipulation of stake, rewards, or access, leading to fund loss, incorrect balances, or denial of service.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

The `RivusCOMAIN::approveMultipleUnstakes` function allows users with the `hasApproveWithdrawalRole` role to approve `requestUnstake` requests and send the requested `wCOMAIN` assets. Users can then call the `RivusCOMAIN::unstake` function to receive the corresponding `wCOMAIN` tokens.

The issue arises because fees can change between the approval of `requestUnstake` and when users perform `unstake`, affecting the actual amount required. Consider the following scenario:

1. A `requestUnstake` for `1 wCOMAI` in exchange for `1 rsCOMAI` is made.
2. The manager approves the request and calculates the `wCOMAI` amount required to be sent to the contract using the `getWCOMAIByrsCOMAIAfterFee` function in line `RivusCOMAI#L724`:

```solidity
File: RivusCOMAI.sol
681:   function approveMultipleUnstakes(UserRequest[] calldata requests)
682:     public
683:     hasApproveWithdrawalRole
684:     nonReentrant
685:     checkPaused
686:   {
...
...
702:     // Loop through each request to unstake and check if the request is valid
703:     for (uint256 i = 0; i < requests.length; i++) {
...
...
724:       (uint256 wcomaiAmt,,uint256 unstakingFeeAmt) = getWCOMAIByrsCOMAIAfterFee(unstakeRequests[request.user][request.requestIndex].comaiAmt);
725:       totalRequiredComaiAmt = totalRequiredComaiAmt + wcomaiAmt + unstakingFeeAmt;
726:     }
...
...
741:     // Transfer the COMAI from the withdrawal manager to this contract
742:     require(
743:       IERC20(commonWrappedToken).transferFrom(
744:         msg.sender,
745:         address(this),
746:         totalRequiredComaiAmt
747:       ),
748:       "comaiAmt transfer failed"
749:     );
...
...
760:   }
```

```solidity
File: RivusCOMAI.sol
476:   function getWCOMAIByrsCOMAIAfterFee(uint256 rsCOMAIAmount)
477:     public
478:     view
479:     returns (uint256, uint256, uint256)
480:   {
481:     uint256 unstakingFeeAmt = rsCOMAIAmount * unstakingFee / 1000;
482:     uint256 bridgingFeeAmt = rsCOMAIAmount * bridgingFee / 1000;
483:     if(bridgingFeeAmt < 1 * (10 ** decimals())) bridgingFeeAmt = 1 * (10 ** decimals());
484:     uint256 unstakingAmt = rsCOMAIAmount - bridgingFeeAmt - unstakingFeeAmt;
485:     return (unstakingAmt, bridgingFeeAmt, unstakingFeeAmt);
486:   }
```

3. Fees are then changed using the `RivusCOMAI::setUnstakingFee` and `RivusCOMAI::setBridgingFee` functions.
4. The user with the approved `requestUnstake` calls the `RivusCOMAI::unstake` function, which again calls the `getWCOMAIByrsCOMAIAfterFee` function in line `RivusCOMAI#L781`, resulting in different fees due to the changes made in `step 3`:

```solidity
File: RivusCOMAI.sol
770:   function unstake(uint256 requestIndex) public nonReentrant checkPaused {
771:
772:     require(
773:       requestIndex < unstakeRequests[msg.sender].length,
774:       "Invalid request index"
775:     );
776:     UnstakeRequest memory request = unstakeRequests[msg.sender][requestIndex];
777:     require(request.amount > 0, "No unstake request found");
778:     require(request.isReadyForUnstake, "Unstake not approved yet");
779:
780:     // Transfer wCOMAI tokens back to the user
781:     (uint256 amountToTransfer,,uint256 unstakingFeeAmt) = getWCOMAIByrsCOMAIAfterFee(request.comaiAmt);
782:     _transferToVault(address(this), unstakingFeeAmt);
783:
784:     // Update state to false
785:     delete unstakeRequests[msg.sender][requestIndex];
786:
787:     // Perform ERC20 transfer
788:     bool transferSuccessful = IERC20(request.wrappedToken).transfer(
789:       msg.sender,
790:       amountToTransfer
791:     );
792:     require(transferSuccessful, "wCOMAI transfer failed");
793:
794:     // Process the unstake event
795:     emit UserUnstake(msg.sender, requestIndex, block.timestamp);
796:   }
```

5. The amount transferred to the user may be incorrect due to the fee changes, leading to insufficient `wCOMAI` deposited by the manager in `step 2`.

This inconsistency can cause some unstake operations to fail because the correct amount of `wCOMAI` was not deposited into the `RivusCOMAI` contract via `approveMultipleUnstakes()`.

## Recommendations

It is recommended to calculate the fees and wrapped token amounts during the `RivusCOMAI::requestUnstake` function, similar to how it is done in `RivusTAO::requestUnstake#L572`. This way, the same `wCOMAI` and `unstakingFees` values are used consistently in both the `approveMultipleUnstakes()` and `unstake()` functions.

---
