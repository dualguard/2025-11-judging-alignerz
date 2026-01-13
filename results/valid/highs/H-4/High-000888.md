# [000888] An owner schedule change will brick or mispay TVS holders
  
  ### Summary

Mutable global vesting parameters combined with sticky per-user claimedSeconds will cause claim reverts or mis-payouts for TVS holders as the owner can change startTime/vestingPeriod after users have begun claiming.

### Root Cause

In `src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:64-118` the owner can mutate `startTime` and `vestingPeriod` at any time, while `claimDividends` (`:152-179`) persists per-user `claimedSeconds` without epoching or resetting; changing these globals mid-stream makes `secondsPassed` negative (revert) or re-bases payouts on a new denominator, corrupting accounting.

### Internal Pre-conditions

1. Owner calls `setStartTime` to move `startTime` forward (later than the original) after users have claimed some dividends.
2. Owner calls `setStartTime` to move `startTime` into the future while dividends are already configured.
3. Owner calls `setVestingPeriod` to change the vest length after some users have partially claimed.

### External Pre-conditions

none

### Attack Path

1. Owner configures dividends and users start claiming, increasing their `claimedSeconds`.
2. Owner adjusts `startTime` to a later timestamp (possibly still in the past to “fix” timing) or into the future.
3. On the next `claimDividends`, `secondsPassed - claimedSeconds` underflows and the call reverts, bricking that user (or everyone if `startTime` is in the future).
4. Alternatively, owner changes `vestingPeriod` mid-stream; subsequent claims use the new denominator with stale `claimedSeconds`, over- or under-paying relative to the original allocation.

### Impact

TVS holders can be globally DoS'd from claiming (if `startTime` is set into the future or reduced below prior `claimedSeconds`) or receive incorrect payouts (over/under payment) after a `vestingPeriod` change; protocol reserves become insufficient or misallocated.

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import {Test, stdError} from "forge-std/Test.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
import {IAlignerzNFT} from "../src/interfaces/IAlignerzNFT.sol";

/// @dev Vesting stub with the minimal fields touched by the distributor
contract VestingStubMutable is IAlignerzVesting {
    mapping(uint256 => Allocation) internal allocations;

    function seedAllocation(
        uint256 nftId,
        IERC20 token,
        uint256[] memory amounts,
        uint256[] memory claimedSeconds,
        uint256[] memory vestingPeriods,
        bool[] memory claimedFlows
    ) external {
        Allocation storage a = allocations[nftId];
        a.token = token;
        a.amounts = amounts;
        a.claimedSeconds = claimedSeconds;
        a.vestingPeriods = vestingPeriods;
        a.claimedFlows = claimedFlows;
    }

    function allocationOf(uint256 nftId) external view returns (Allocation memory) {
        return allocations[nftId];
    }
}

/// @dev NFT stub satisfying the distributor ownership checks
contract NFTStubMutable is IAlignerzNFT {
    uint256 internal totalMinted;
    mapping(uint256 => address) internal ownerOf;

    function mint(address to) external returns (uint256) {
        uint256 tokenId = totalMinted;
        ownerOf[tokenId] = to;
        totalMinted++;
        return tokenId;
    }

    function extOwnerOf(uint256 tokenId) external view returns (address) {
        address owner = ownerOf[tokenId];
        require(owner != address(0), "nonexistent");
        return owner;
    }

    function burn(uint256 tokenId) external {
        ownerOf[tokenId] = address(0);
    }

    function getTotalMinted() external view returns (uint256) {
        return totalMinted;
    }
}

/// @notice Demonstrates schedule mutation breaking or skewing dividend claims
contract A26ZDividendDistributorMutableScheduleTest is Test {
    MockUSD internal stablecoin;
    MockUSD internal vestingToken;
    VestingStubMutable internal vesting;
    NFTStubMutable internal nft;
    A26ZDividendDistributor internal distributor;

    address internal user = address(0xBEEF);
    uint256 internal constant DEFAULT_VESTING_PERIOD = 100;

    function setUp() public {
        stablecoin = new MockUSD();
        vestingToken = new MockUSD();
        vesting = new VestingStubMutable();
        nft = new NFTStubMutable();

        distributor = new A26ZDividendDistributor(
            address(vesting), address(nft), address(stablecoin), block.timestamp, DEFAULT_VESTING_PERIOD, address(0xDEAD)
        );
    }

    function _mintAndSeed(uint256 amount) internal returns (uint256 nftId) {
        nftId = nft.mint(user);
        uint256[] memory amounts = new uint256[](1);
        uint256[] memory claimedSeconds = new uint256[](1);
        uint256[] memory vestingPeriods = new uint256[](1);
        bool[] memory claimedFlows = new bool[](1);

        amounts[0] = amount;
        claimedSeconds[0] = 1; // keep non-zero to avoid the infinite loop path in getUnclaimedAmounts
        vestingPeriods[0] = 1e9; // make claimedAmount zero so unclaimed == amount
        claimedFlows[0] = false;

        vesting.seedAllocation(nftId, IERC20(address(vestingToken)), amounts, claimedSeconds, vestingPeriods, claimedFlows);
    }

    function _fundAndSetup(uint256 amount) internal {
        stablecoin.transfer(address(distributor), amount);
        distributor.setUpTheDividends();
    }

    function test_claimRevertsWhenOwnerPushesStartTimeIntoFuture() public {
        _mintAndSeed(100e6);
        _fundAndSetup(100e6);

        distributor.setStartTime(block.timestamp + 1 days);

        vm.expectRevert(stdError.arithmeticError);
        vm.prank(user);
        distributor.claimDividends();
    }

    function test_startTimeShiftForwardBricksExistingClaimant() public {
        _mintAndSeed(100e6);
        _fundAndSetup(100e6);

        uint256 initialStart = distributor.startTime();

        vm.warp(initialStart + 10);
        vm.prank(user);
        distributor.claimDividends(); // claimedSeconds now 10

        // Move start time forward but keep it in the past; next claim underflows on claimedSeconds delta
        distributor.setStartTime(initialStart + 5);

        vm.expectRevert(stdError.arithmeticError);
        vm.prank(user);
        distributor.claimDividends();
    }

    function test_vestingPeriodChangeOverpaysUser() public {
        _mintAndSeed(100e6);
        _fundAndSetup(100e6);

        uint256 initialStart = distributor.startTime();

        // First partial claim at half the original schedule
        vm.warp(initialStart + DEFAULT_VESTING_PERIOD / 2);
        vm.prank(user);
        distributor.claimDividends(); // pays 50e6, stores claimedSeconds = 50

        // Top up to allow the contract to cover the eventual overpayment
        stablecoin.transfer(address(distributor), 100e6);

        // Extend the vesting period mid-stream: payout uses new denominator but old claimedSeconds
        distributor.setVestingPeriod(200);

        vm.warp(initialStart + 200);
        vm.prank(user);
        distributor.claimDividends(); // pays 75e6, total 125e6 on a 100e6 allocation

        assertEq(distributor.stablecoinAmountToDistribute(), 100e6, "original schedule was funded for 100e6");
        assertEq(stablecoin.balanceOf(user), 125e6, "user is overpaid after vestingPeriod mutation");
    }
}
```
- `test_claimRevertsWhenOwnerPushesStartTimeIntoFuture` shows global DoS when `startTime` is moved to the future (underflow revert).
- `test_startTimeShiftForwardBricksExistingClaimant` shows per-user permanent DoS when `startTime` is shifted forward after partial claims (underflow revert on delta).
- `test_vestingPeriodChangeOverpaysUser` shows overpayment when `vestingPeriod` is extended after partial claims (user receives 125% of allocation).

### Mitigation

- Lock `startTime`/`vestingPeriod` after initialization, or only allow changes before any claim has occurred.
- If schedule changes are necessary, introduce an epoch version and reset/recompute per-user progress on change (or store per-epoch claimedSeconds).
- Alternatively, prohibit decreases/increases that would make `secondsPassed < claimedSeconds`, and re-derive claim state from first principles when parameters change.
  