# [000691] Missing Storage Gap in Upgradeable Contract AlignerzVesting
  
  ### Summary

The missing storage gap (`__gap`) in `AlignerzVesting.sol` will cause complete state corruption for all users as protocol developers will inadvertently shift storage slots when adding new state variables during a contract upgrade.

### Root Cause

In [`AlignerzVesting.sol:18`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L18), the contract inherits from upgradeable contracts (`OwnableUpgradeable`, `WhitelistManager`, `FeesManager`) that all properly define storage gaps, but `AlignerzVesting` itself does not define a `__gap` array at the end of its state variables (lines 84-114).

This violates OpenZeppelin's upgrade safety guidelines. When new state variables are added to `AlignerzVesting` in a future upgrade, they will shift the storage layout and cause all existing state to be read from incorrect slots.

The parent contracts properly implement gaps:
- `FeesManager.sol:180`: `uint256[50] private __gap;`
- `WhitelistManager.sol:147`: `uint256[50] private __gap;`
- `AlignerzVesting.sol`: NO GAP DEFINED

### Internal Pre-conditions

1. Protocol developers need to deploy an upgrade to `AlignerzVesting` that adds new state variables
2. The new state variables need to be declared before the existing state variables in the upgraded contract (common mistake when extending functionality)

### External Pre-conditions

None required. This is purely an internal contract upgrade issue.

### Attack Path

1. Protocol deploys `AlignerzVesting` V1 with existing state variables (`biddingProjectCount`, `treasury`, `vestingPeriodDivisor`, etc.)
2. Users interact with the protocol: create projects, place bids, claim NFTs, set treasury address
3. Protocol developers create `AlignerzVesting` V2 that adds new state variables (e.g., `newFeatureFlag`, `pauseStatus`) to extend functionality
4. Protocol calls `upgradeToAndCall()` to upgrade to V2
5. All state variables shift by N slots (where N is the number of new variables added)
6. `biddingProjectCount` now reads from the slot that previously held `vestingPeriodDivisor` (value: 2,592,000)
7. `treasury` now reads from a garbage slot (value: `address(0)`)
8. All protocol operations fail or behave unexpectedly due to corrupted state

### Impact

The protocol and all users suffer complete loss of functionality and potential loss of funds:

1. Treasury Corruption: Treasury address becomes `address(0)`, causing:
   - Fee transfers in `mergeTVS()` and `splitTVS()` to revert or burn tokens
   - `withdrawPostDeadlineProfit()` to send funds to `address(0)`
   - All fee-related operations to fail

2. Project Count Corruption: `biddingProjectCount` reads incorrect value (e.g., 2,592,000 instead of actual count), causing:
   - New project launches to potentially overwrite existing projects
   - Project ID collisions and data loss

3. Complete State Corruption: All mappings and state variables become misaligned, rendering the contract non-functional

4. User Fund Loss: Users cannot claim tokens, refunds, or interact with their allocations due to corrupted state

### PoC

Save this POC to 


Run the test:
```bash
forge test --match-path test/M04_MissingStorageGap.t.sol -vvv
```

### POC Code

```
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {WhitelistManager} from "../src/contracts/vesting/whitelistManager/WhitelistManager.sol";
import {FeesManager} from "../src/contracts/vesting/feesManager/FeesManager.sol";
import {IAlignerzNFT} from "../src/interfaces/IAlignerzNFT.sol";

// V2 contract that simulates adding new state variables during an upgrade.
// Without a storage gap in V1, adding variables shifts the storage layout
// and causes all subsequent variables to read from wrong slots.
//
// Storage layout comparison:
// V1 (AlignerzVesting):          V2 (with new variables):
// slot X:   biddingProjectCount  slot X:   newFeatureFlag      <- COLLISION
// slot X+1: rewardProjectCount   slot X+1: anotherNewVariable  <- COLLISION
// slot X+2: vestingPeriodDivisor slot X+2: biddingProjectCount <- reads old vestingPeriodDivisor!
// slot X+3: treasury             slot X+3: rewardProjectCount  <- reads old treasury!
// slot X+4: nftContract          slot X+4: vestingPeriodDivisor<- reads old nftContract!
// slot X+5: ...                  slot X+5: treasury            <- reads garbage/0!
contract AlignerzVestingV2StorageCollision is Initializable, UUPSUpgradeable, OwnableUpgradeable, WhitelistManager, FeesManager {
    using SafeERC20 for IERC20;

    struct RewardProject {
        IERC20 token;
        IERC20 stablecoin;
        uint256 vestingPeriod;
        mapping(uint256 => Allocation) allocations;
        mapping(address => uint256) kolTVSRewards;
        mapping(address => uint256) kolStablecoinRewards;
        mapping(address => uint256) kolTVSIndexOf;
        mapping(address => uint256) kolStablecoinIndexOf;
        address[] kolTVSAddresses;
        address[] kolStablecoinAddresses;
        uint256 startTime;
        uint256 claimDeadline;
    }

    struct BiddingProject {
        IERC20 token;
        IERC20 stablecoin;
        uint256 totalStablecoinBalance;
        uint256 poolCount;
        uint256 startTime;
        uint256 endTime;
        mapping(uint256 => VestingPool) vestingPools;
        mapping(address => Bid) bids;
        mapping(uint256 => Allocation) allocations;
        bytes32 refundRoot;
        bytes32 endTimeHash;
        bool closed;
        uint256 claimDeadline;
    }

    struct VestingPool {
        bytes32 merkleRoot;
        bool hasExtraRefund;
    }

    struct Bid {
        uint256 amount;
        uint256 vestingPeriod;
    }

    struct Allocation {
        uint256[] amounts;
        uint256[] vestingPeriods;
        uint256[] vestingStartTimes;
        uint256[] claimedSeconds;
        bool[] claimedFlows;
        bool isClaimed;
        IERC20 token;
        uint256 assignedPoolId;
    }

    // NEW VARIABLES - these shift all subsequent storage slots by 2
    uint256 public newFeatureFlag;
    uint256 public anotherNewVariable;

    // ORIGINAL VARIABLES - now at shifted slots
    uint256 public biddingProjectCount;
    uint256 public rewardProjectCount;
    uint256 public vestingPeriodDivisor;
    address public treasury;
    IAlignerzNFT public nftContract;
    mapping(uint256 => BiddingProject) public biddingProjects;
    mapping(uint256 => RewardProject) public rewardProjects;
    mapping(bytes32 => bool) public claimedRefund;
    mapping(bytes32 => bool) public claimedNFT;
    mapping(uint256 => bool) public NFTBelongsToBiddingProject;
    mapping(uint256 => Allocation) public allocationOf;

    error Zero_Address();

    function initializeV2() public reinitializer(2) {
        newFeatureFlag = 12345;
        anotherNewVariable = 67890;
    }

    function _authorizeUpgrade(address) internal override onlyOwner {}
}

contract M04_MissingStorageGapTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;
    address public owner;
    address payable public proxy;

    uint256 constant TOKEN_AMOUNT = 26_000_000 ether;

    function setUp() public {
        owner = address(this);

        usdt = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        AlignerzVesting implementation = new AlignerzVesting();
        bytes memory initData = abi.encodeCall(AlignerzVesting.initialize, (address(nft)));
        proxy = payable(address(new ERC1967Proxy(address(implementation), initData)));

        vesting = AlignerzVesting(proxy);
        nft.addMinter(proxy);
        vesting.setTreasury(address(0xDEAD));
        token.approve(address(vesting), TOKEN_AMOUNT);
    }

    function test_StorageCollisionAfterUpgrade() public {
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1_000_000,
            bytes32(0),
            false
        );

        uint256 projectCountBefore = vesting.biddingProjectCount();
        address treasuryBefore = vesting.treasury();
        uint256 vestingDivisorBefore = vesting.vestingPeriodDivisor();

        assertEq(projectCountBefore, 1);
        assertEq(treasuryBefore, address(0xDEAD));
        assertEq(vestingDivisorBefore, 2_592_000);

        AlignerzVestingV2StorageCollision newImpl = new AlignerzVestingV2StorageCollision();
        vm.prank(owner);
        AlignerzVesting(proxy).upgradeToAndCall(
            address(newImpl),
            abi.encodeCall(AlignerzVestingV2StorageCollision.initializeV2, ())
        );

        AlignerzVestingV2StorageCollision vestingV2 = AlignerzVestingV2StorageCollision(proxy);

        uint256 projectCountAfter = vestingV2.biddingProjectCount();
        address treasuryAfter = vestingV2.treasury();
        uint256 vestingDivisorAfter = vestingV2.vestingPeriodDivisor();

        emit log_string("BEFORE UPGRADE (V1)");
        emit log_named_uint("biddingProjectCount", projectCountBefore);
        emit log_named_address("treasury", treasuryBefore);
        emit log_named_uint("vestingPeriodDivisor", vestingDivisorBefore);

        emit log_string("AFTER UPGRADE (V2) - Storage Corrupted");
        emit log_named_uint("biddingProjectCount (CORRUPTED)", projectCountAfter);
        emit log_named_address("treasury (CORRUPTED)", treasuryAfter);
        emit log_named_uint("vestingPeriodDivisor (CORRUPTED)", vestingDivisorAfter);

        assertEq(projectCountAfter, vestingDivisorBefore, "biddingProjectCount now reads old vestingPeriodDivisor slot");
        assertTrue(treasuryAfter != treasuryBefore, "treasury now reads from wrong slot");
    }

    function test_StorageSlotShiftExplanation() public {
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1_000_000,
            bytes32(0),
            false
        );
        vesting.setTreasury(address(0xBEEF));

        uint256 v1ProjectCount = vesting.biddingProjectCount();
        uint256 v1VestingDivisor = vesting.vestingPeriodDivisor();
        address v1Treasury = vesting.treasury();

        emit log_string("=== V1 Storage Layout ===");
        emit log_named_uint("slot X:   biddingProjectCount", v1ProjectCount);
        emit log_named_uint("slot X+2: vestingPeriodDivisor", v1VestingDivisor);
        emit log_named_address("slot X+3: treasury", v1Treasury);

        AlignerzVestingV2StorageCollision newImpl = new AlignerzVestingV2StorageCollision();
        vm.prank(owner);
        AlignerzVesting(proxy).upgradeToAndCall(
            address(newImpl),
            abi.encodeCall(AlignerzVestingV2StorageCollision.initializeV2, ())
        );

        AlignerzVestingV2StorageCollision vestingV2 = AlignerzVestingV2StorageCollision(proxy);

        emit log_string("=== V2 Storage Layout (shifted by 2 slots) ===");
        emit log_named_uint("slot X:   newFeatureFlag (set by initializeV2)", vestingV2.newFeatureFlag());
        emit log_named_uint("slot X+1: anotherNewVariable (set by initializeV2)", vestingV2.anotherNewVariable());
        emit log_named_uint("slot X+2: biddingProjectCount (reads V1's vestingPeriodDivisor!)", vestingV2.biddingProjectCount());
        emit log_named_address("slot X+5: treasury (reads garbage slot!)", vestingV2.treasury());

        assertEq(
            vestingV2.biddingProjectCount(),
            v1VestingDivisor,
            "V2.biddingProjectCount reads from V1.vestingPeriodDivisor slot"
        );
    }

    function test_ImpactAnalysis_FundsAtRisk() public {
        token.transfer(address(vesting), 1_000_000 ether);

        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1_000_000,
            bytes32(0),
            false
        );
        vesting.setTreasury(address(0xCAFE));

        AlignerzVestingV2StorageCollision newImpl = new AlignerzVestingV2StorageCollision();
        vm.prank(owner);
        AlignerzVesting(proxy).upgradeToAndCall(
            address(newImpl),
            abi.encodeCall(AlignerzVestingV2StorageCollision.initializeV2, ())
        );

        AlignerzVestingV2StorageCollision vestingV2 = AlignerzVestingV2StorageCollision(proxy);
        address corruptedTreasury = vestingV2.treasury();

        emit log_named_address("Expected Treasury", address(0xCAFE));
        emit log_named_address("Corrupted Treasury", corruptedTreasury);

        assertTrue(corruptedTreasury == address(0), "Treasury corrupted to address(0)");
    }

}

```

### Mitigation

Add a storage gap at the end of the `AlignerzVesting` contract's state variable declarations (after line 114):

```solidity
/// @notice Mapping to fetch the allocation of a TVS given the NFT Id
mapping(uint256 => Allocation) public allocationOf;

/// @dev Reserved storage gap for future upgrades
uint256[50] private __gap;
```

This reserves 50 storage slots that can be consumed by future state variables without shifting the storage layout of any contracts that inherit from `AlignerzVesting`.

  