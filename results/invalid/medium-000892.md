# [000892] Owner rerun setup will over-allocate dividends and bankrupt distributor
  
  ### Summary

Lack of reset/epoching in dividend setup will cause over-allocation, insolvency, and reverted claims for TVS holders as the owner (or an ops script) will rerun `setUpTheDividends`/`setDividends` and compound prior principals instead of replacing them..

### Root Cause

In `vesting.sol:55` there is a missing check for ownership on a function that should be admin only"
      value: "In `src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:197-215` `_setDividends` uses `+=` to add allocations to `dividendsOf[owner].amount` and never clears state; `setUpTheDividends`/`setDividends` are callable repeatedly, so reruns stack new principals on top of old ones using the full contract balance each time.

### Internal Pre-conditions
1. Owner funds the distributor and runs `setUpTheDividends` (or `setAmounts` + `setDividends`) once.
2. Owner reruns `setUpTheDividends`/`setDividends` in the same distribution window (e.g., after topping up stablecoin, retrying, or after NFT ownership changes) without clearing state.

### External Pre-conditions

none

### Attack Path

1. Owner sets up dividends and credits users based on current balance and snapshots.
        2. Owner reruns `setUpTheDividends` (or just `setDividends`), which adds a fresh full allocation to existing `dividendsOf` entries using the current balance.
        3. Credited principals now exceed available stablecoin; if NFTs changed hands, both old and new owners may be credited.
        4. At/after vest end, claims either overpay early callers or revert for insufficient balance, blocking remaining users.

### Impact

TVS holders can be overpaid beyond the funded distribution, draining the pool, or see claims revert for insufficient balance after over-allocation; ownership changes can duplicate credits across old/new holders.

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {IERC20Errors} from "@openzeppelin/contracts/interfaces/draft-IERC6093.sol";
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
import {IAlignerzNFT} from "../src/interfaces/IAlignerzNFT.sol";

/// @dev Vesting stub with minimal fields used by the distributor
contract VestingStubRepeat is IAlignerzVesting {
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

/// @dev NFT stub with mutable ownership for re-snapshot scenarios
contract NFTStubRepeat is IAlignerzNFT {
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

    function transferOwner(uint256 tokenId, address to) external {
        ownerOf[tokenId] = to;
    }

    function getTotalMinted() external view returns (uint256) {
        return totalMinted;
    }
}

/// @notice Demonstrates how repeated setup compounds payouts and drains/blocks the pool
contract A26ZDividendDistributorRepeatedSetupTest is Test {
    MockUSD internal stablecoin;
    MockUSD internal vestingToken;
    VestingStubRepeat internal vesting;
    NFTStubRepeat internal nft;
    A26ZDividendDistributor internal distributor;

    address internal user = address(0xBEEF);
    address internal user2 = address(0xCAFE);

    function setUp() public {
        stablecoin = new MockUSD();
        vestingToken = new MockUSD();
        vesting = new VestingStubRepeat();
        nft = new NFTStubRepeat();

        distributor = new A26ZDividendDistributor(
            address(vesting), address(nft), address(stablecoin), block.timestamp, 100, address(0xDEAD)
        );
    }

    function _mintAndSeed(uint256 amount) internal returns (uint256 nftId) {
        nftId = nft.mint(user);

        uint256[] memory amounts = new uint256[](1);
        uint256[] memory claimedSeconds = new uint256[](1);
        uint256[] memory vestingPeriods = new uint256[](1);
        bool[] memory claimedFlows = new bool[](1);

        amounts[0] = amount;
        claimedSeconds[0] = 1; // avoid the claimedSeconds==0 infinite-loop path
        vestingPeriods[0] = 1e9; // keep unclaimed == amount
        claimedFlows[0] = false;

        vesting.seedAllocation(nftId, IERC20(address(vestingToken)), amounts, claimedSeconds, vestingPeriods, claimedFlows);
    }

    function _fund(uint256 amount) internal {
        stablecoin.transfer(address(distributor), amount);
    }

    function test_setUpTheDividends_compoundsAndRevertsOnPayout() public {
        _mintAndSeed(100e6);
        _fund(100e6);

        distributor.setUpTheDividends();
        distributor.setUpTheDividends(); // second run doubles credited amount

        vm.warp(distributor.startTime() + distributor.vestingPeriod());
        vm.prank(user);
        vm.expectRevert(
            abi.encodeWithSelector(
                IERC20Errors.ERC20InsufficientBalance.selector, address(distributor), 100e6, 200e6
            )
        );
        distributor.claimDividends();
    }

    function test_setDividends_alone_compoundsOnStaleSnapshot() public {
        _mintAndSeed(100e6);
        _fund(100e6);

        distributor.setAmounts();
        distributor.setDividends();
        distributor.setDividends(); // rerun without resetting state

        vm.warp(distributor.startTime() + distributor.vestingPeriod());
        vm.prank(user);
        vm.expectRevert(
            abi.encodeWithSelector(
                IERC20Errors.ERC20InsufficientBalance.selector, address(distributor), 100e6, 200e6
            )
        );
        distributor.claimDividends();
    }

    function test_rerunAfterOwnershipChange_doubleCreditsNewOwner_blocksOldOwner() public {
        uint256 nftId = _mintAndSeed(100e6);
        _fund(100e6);

        distributor.setUpTheDividends(); // credits user with 100e6

        // Transfer NFT to a new owner without recomputing unclaimedAmountsIn
        nft.transferOwner(nftId, user2);
        distributor.setDividends(); // credits user2 using stale snapshot while user keeps old credit

        vm.warp(distributor.startTime() + distributor.vestingPeriod());

        // New owner drains the pool
        vm.prank(user2);
        distributor.claimDividends();

        // Original owner is now bricked due to over-allocation
        vm.prank(user);
        vm.expectRevert(
            abi.encodeWithSelector(
                IERC20Errors.ERC20InsufficientBalance.selector, address(distributor), 0, 100e6
            )
        );
        distributor.claimDividends();
    }
}
```

### Mitigation

- Replace `+=` with assignment after clearing/dividing state: reset `dividendsOf` and `unclaimedAmountsIn` before recomputing.
- Consider single-use setup or an epoch/versioned snapshot so reruns overwrite, not compound.
- Optionally track already-claimed amounts separately to prevent new principals from being instantly claimable when vesting is over.
  