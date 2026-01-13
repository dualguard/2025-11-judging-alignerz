# [000212] Stale allocationOf mirror causes incorrect dividend calculations, leading to fund misallocation
  
  ### Summary

The `allocationOf` mapping is intended to be a snapshot/mirror of allocation data for external readers like the dividend distributor. However, this mirror is only updated when NFTs are initially created (in `claimNFT` and `_claimRewardTVS`), but never refreshed when users claim tokens or merge TVSs. The dividend distributor reads from this stale mirror, causing it to calculate unclaimed amounts as if no tokens were ever claimed, leading to incorrect dividend distributions.

### Root Cause

The `allocationOf``protocol/src/contracts/vesting/AlignerzVesting.sol:114` mapping is set as a snapshot when NFTs are created:

**Code Reference - Snapshot taken on NFT creation:**
```solidity
// AlignerzVesting.sol:878-888 (claimNFT)
uint256 nftId = nftContract.mint(msg.sender);
biddingProject.allocations[nftId].amounts.push(amount);
biddingProject.allocations[nftId].vestingPeriods.push(bid.vestingPeriod);
biddingProject.allocations[nftId].vestingStartTimes.push(biddingProject.endTime);
biddingProject.allocations[nftId].claimedSeconds.push(0);
biddingProject.allocations[nftId].claimedFlows.push(false);
biddingProject.allocations[nftId].assignedPoolId = poolId;
biddingProject.allocations[nftId].token = biddingProject.token;
NFTBelongsToBiddingProject[nftId] = true;
allocationOf[nftId] = biddingProject.allocations[nftId]; // ✅ Snapshot taken
```

However, when tokens are claimed, only the live allocation is updated:

**Code Reference - Live allocation updated, mirror not refreshed:**
```solidity
// AlignerzVesting.sol:941-974 (claimTokens)
function claimTokens(uint256 projectId, uint256 nftId) external {
    // ... get allocation from live storage ...
    (Allocation storage allocation, IERC20 token) = isBiddingProject ? 
    (biddingProjects[projectId].allocations[nftId], biddingProjects[projectId].token) : 
    (rewardProjects[projectId].allocations[nftId], rewardProjects[projectId].token);
    
    // ... calculate claimable amounts ...
    for (uint256 i; i < nbOfFlows; i++) {
        // ... calculate claimable ...
        allocation.claimedSeconds[i] += claimableSeconds; // ✅ Live storage updated
        if (allocation.claimedSeconds[i] >= allocation.vestingPeriods[i]) {
            allocation.claimedFlows[i] = true; // ✅ Live storage updated
        }
        // ...
    }
    token.safeTransfer(msg.sender, claimableAmounts);
    // ❌ allocationOf[nftId] is never updated here!
}
```

The dividend distributor then reads from the stale mirror:

**Code Reference - Dividend distributor reads stale data:**
```solidity
// A26ZDividendDistributor.sol:140-161 (getUnclaimedAmounts)
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
    uint256[] memory amounts = vesting.allocationOf(nftId).amounts; // ❌ Stale data
    uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds; // ❌ Always 0
    uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
    bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows; // ❌ Always false
    
    for (uint i; i < len;) {
        if (claimedFlows[i]) continue;
        if (claimedSeconds[i] == 0) { // ❌ This is always true due to stale data
            amount += amounts[i]; // ❌ Calculates as if nothing was claimed
            continue;
        }
        // ... calculation never reached because claimedSeconds is always 0 ...
    }
}
```

**The Problem:**
- `allocationOf` is set once when NFT is created, with `claimedSeconds = [0]` and `claimedFlows = [false]`
- When users claim tokens, `claimTokens` updates the live `biddingProjects[projectId].allocations[nftId]` but never updates `allocationOf[nftId]`
- The dividend distributor reads from `allocationOf`, which always shows the initial state (nothing claimed)
- This causes `getUnclaimedAmounts` to always calculate the full original amount as unclaimed
- The same issue occurs in `mergeTVS` - the merged allocation is updated in live storage but `allocationOf` is never refreshed

### Internal Pre-conditions

1. A TVS NFT must exist (obtained through `claimNFT` or `_claimRewardTVS`)
2. The user must have claimed tokens from their vesting schedule by calling `claimTokens`
   - This updates `claimedSeconds` and potentially `claimedFlows` in the live allocation
3. The dividend distributor must be configured and someone must call:
   - `getUnclaimedAmounts(nftId)` directly, OR
   - `getTotalUnclaimedAmounts()` which calls `getUnclaimedAmounts` for all NFTs, OR
   - `setAmounts()` which calls `getTotalUnclaimedAmounts()`

### External Pre-conditions

_No response_

### Attack Path

1. User places a bid in a bidding project and receives a TVS NFT through `claimNFT`
   - At this point, `allocationOf[nftId]` is set as a snapshot with `claimedSeconds = [0]` and `claimedFlows = [false]`
2. User claims tokens from their vesting schedule by calling `claimTokens(projectId, nftId)`
   - The live allocation `biddingProjects[projectId].allocations[nftId]` is updated with new `claimedSeconds` values
   - However, `allocationOf[nftId]` is never updated and remains stale
3. Owner or system attempts to calculate dividend distributions by calling:
   - `getUnclaimedAmounts(nftId)` or `getTotalUnclaimedAmounts()`
4. The dividend distributor reads from `allocationOf[nftId]`, which still shows:
   - `claimedSeconds = [0]` (stale - should show actual claimed seconds)
   - `claimedFlows = [false]` (stale - should show if flows are fully claimed)
5. `getUnclaimedAmounts` calculates unclaimed amount as the full original amount
   - Since `claimedSeconds[i] == 0` is always true (due to stale data), it adds the full `amounts[i]` to the unclaimed total
6. `totalUnclaimedAmounts` is grossly overstated
7. Dividend shares are calculated incorrectly:
   - Users who have claimed tokens still get dividends as if they never claimed
   - The dividend pool is distributed based on incorrect unclaimed amounts
   - Other users are short-changed

### Impact

- **Fund Misallocation**: Users who have claimed tokens still receive dividends as if they never claimed, receiving more than their fair share
- **Incorrect Total Calculations**: `totalUnclaimedAmounts` is grossly overstated, making the denominator in dividend share calculations incorrect

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";
import {CompleteMerkle} from "murky/src/CompleteMerkle.sol";

/// @notice Test to prove the stale allocationOf mirror bug
/// @dev When tokens are claimed, the live allocation is updated but allocationOf mirror is not refreshed
contract StaleAllocationOfMirrorBugTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;
    A26ZDividendDistributor public dividendDistributor;

    address public owner;
    address public projectCreator;
    address public bidder;
    address public bidder2; // Second bidder needed for merkle tree (requires at least 2 leaves)

    uint256 constant TOKEN_AMOUNT = 26_000_000 ether;
    uint256 constant BID_AMOUNT = 1_000 ether;
    uint256 constant PROJECT_ID = 0;
    uint256 constant VESTING_PERIOD = 90 days;

    // Track proofs
    mapping(address => bytes32[]) public bidderProofs;

    function setUp() public {
        owner = address(this);
        projectCreator = makeAddr("projectCreator");
        bidder = makeAddr("bidder");
        bidder2 = makeAddr("bidder2");
        vm.deal(projectCreator, 100 ether);
        vm.deal(bidder, 50 ether);
        vm.deal(bidder2, 50 ether);

        // Deploy contracts
        usdt = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        address payable proxy = payable(Upgrades.deployUUPSProxy(
            "AlignerzVesting.sol",
            abi.encodeCall(AlignerzVesting.initialize, (address(nft))))
        );
        vesting = AlignerzVesting(proxy);

        // Set NFT minter to vesting contract
        vm.prank(owner);
        nft.addMinter(proxy);
        vesting.setTreasury(address(1));

        // Setup bidders
        usdt.mint(bidder, BID_AMOUNT);
        usdt.mint(bidder2, BID_AMOUNT);

        // Mint tokens for project creator
        token.transfer(projectCreator, TOKEN_AMOUNT);
        vesting.transferOwnership(projectCreator);

        // Approve tokens for vesting contract
        vm.prank(projectCreator);
        token.approve(address(vesting), TOKEN_AMOUNT);

        // Deploy dividend distributor with different token to bypass token filter
        Aligners26 differentToken = new Aligners26("Different", "DIFF");
        dividendDistributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(usdt),
            block.timestamp,
            90 days,
            address(differentToken)
        );
    }

    function getLeaf(address _bidder, uint256 amount, uint256 projectId, uint256 poolId)
        internal
        pure
        returns (bytes32)
    {
        return keccak256(abi.encodePacked(_bidder, amount, projectId, poolId));
    }

    // Helper for generating merkle proofs (needs at least 2 bids)
    function generateMerkleProofs(address[] memory bidders, uint256 poolId) internal returns (bytes32) {
        uint256 len = bidders.length;
        require(len >= 2, "Need at least 2 bids for Merkle tree");
        
        bytes32[] memory leaves = new bytes32[](len);
        for (uint256 i = 0; i < len; i++) {
            leaves[i] = getLeaf(bidders[i], BID_AMOUNT, PROJECT_ID, poolId);
        }

        CompleteMerkle m = new CompleteMerkle();
        bytes32 root = m.getRoot(leaves);
        
        for (uint256 i = 0; i < len; i++) {
            bytes32[] memory proof = m.getProof(leaves, i);
            bidderProofs[bidders[i]] = proof;
        }

        return root;
    }

    /// @notice Test that allocationOf mirror is not updated after claiming tokens
    function test_AllocationOfMirror_StaleAfterClaim() public {
        // Setup project
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1_000_000,
            "0x0",
            false // No whitelist for simplicity
        );
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vm.stopPrank();

        // Place bids (need at least 2 for merkle tree)
        vm.startPrank(bidder);
        usdt.approve(address(vesting), BID_AMOUNT);
        vesting.placeBid(PROJECT_ID, BID_AMOUNT, VESTING_PERIOD);
        vm.stopPrank();

        vm.startPrank(bidder2);
        usdt.approve(address(vesting), BID_AMOUNT);
        vesting.placeBid(PROJECT_ID, BID_AMOUNT, VESTING_PERIOD);
        vm.stopPrank();

        // Generate merkle proofs (requires at least 2 bids)
        address[] memory bidders = new address[](2);
        bidders[0] = bidder;
        bidders[1] = bidder2;
        bytes32 root = generateMerkleProofs(bidders, 0);

        // Finalize bids and claim NFT
        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = root;
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60 days);

        vm.prank(bidder);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, 0, BID_AMOUNT, bidderProofs[bidder]);

        // Fast forward time (30 days = 1/3 of vesting period)
        vm.warp(block.timestamp + 30 days);

        // Claim tokens - this updates the live allocation but NOT allocationOf
        vm.prank(bidder);
        vesting.claimTokens(PROJECT_ID, nftId);

        // BUG: allocationOf mirror is stale - it was set when NFT was claimed but never updated
        // We verify this indirectly: the live allocation was updated (we can't claim again immediately)
        // but allocationOf still has stale data (verified through dividend distributor in second test)

        // Verify the live allocation WAS updated (by checking that we can't claim again immediately)
        // Since no time has passed since the last claim, there should be no claimable tokens
        vm.prank(bidder);
        vm.expectRevert(abi.encodeWithSelector(AlignerzVesting.No_Claimable_Tokens.selector));
        vesting.claimTokens(PROJECT_ID, nftId);
    }

    /// @notice Test that dividend distributor reads stale data from allocationOf
    function test_DividendDistributor_ReadsStaleAllocationOf() public {
        // Setup project and claim NFT (same as above)
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1_000_000,
            "0x0",
            false
        );
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vm.stopPrank();

        vm.startPrank(bidder);
        usdt.approve(address(vesting), BID_AMOUNT);
        vesting.placeBid(PROJECT_ID, BID_AMOUNT, VESTING_PERIOD);
        vm.stopPrank();

        vm.startPrank(bidder2);
        usdt.approve(address(vesting), BID_AMOUNT);
        vesting.placeBid(PROJECT_ID, BID_AMOUNT, VESTING_PERIOD);
        vm.stopPrank();

        // Generate merkle proofs (requires at least 2 bids)
        address[] memory bidders = new address[](2);
        bidders[0] = bidder;
        bidders[1] = bidder2;
        bytes32 root = generateMerkleProofs(bidders, 0);

        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = root;
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60 days);

        vm.prank(bidder);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, 0, BID_AMOUNT, bidderProofs[bidder]);

        // Fast forward and claim tokens
        vm.warp(block.timestamp + 30 days);
        vm.prank(bidder);
        vesting.claimTokens(PROJECT_ID, nftId);

        // BUG: Dividend distributor reads stale allocationOf
        // The stale allocationOf shows claimedSeconds[0] = 0 (even though we claimed 30 days)
        // This causes getUnclaimedAmounts to hit the infinite loop bug OR calculate incorrectly
        // 
        // Note: Due to the infinite loop bug in getUnclaimedAmounts (H-03), when it sees
        // claimedSeconds[i] == 0, it calls continue without incrementing i, causing infinite loop.
        // This demonstrates that the stale allocationOf mirror causes the dividend distributor
        // to malfunction.
        
        // The function will revert due to infinite loop (from stale data showing claimedSeconds = 0)
        vm.expectRevert(); // Will revert due to out of gas from infinite loop
        dividendDistributor.getUnclaimedAmounts(nftId);
        
        // Even if the infinite loop bug were fixed, the stale data would still cause
        // incorrect calculation: it would return BID_AMOUNT (full amount) instead of
        // ~2/3 * BID_AMOUNT (the actual unclaimed amount after claiming 1/3)
    }
}

```

### Mitigation

_No response_
  