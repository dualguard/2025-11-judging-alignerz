# [000734] Unimplemented hasExtraRefund Logic Will Cause Loss of Excess Funds for Winning Bidders
  
  ### Summary

The `hasExtraRefund` boolean flag in the `VestingPool` struct is set during pool creation but never used in any contract logic This will cause winning bidders in pools with`hasExtraRefund = true` to lose their excess stablecoin as the protocol owner will withdraw these funds via `withdrawPostDeadlineProfit` instead of refunding them to users who may have overbid relative to their final allocation
```solidity
 /// @notice Represents a vesting pool within a biddingProject
    /// @dev Contains allocation and vesting parameters for a specific pool
    struct VestingPool {
        bytes32 merkleRoot; // Merkle root for allocated bids
        bool hasExtraRefund; // whether pool refunds the winners as well //@> Never used
    }
```

### Root Cause

In `AlignerzVesting::createPool`, the function accepts a `hasExtraRefund` parameter and stores it in the `VestingPool` struct. However, the `claimNFT` function never reads this flag to calculate or transfer any excess refund to winning bidders.  The flag is defined but completely unused in the contract's business logic.
```solidity
 function claimNFT(uint256 projectId, uint256 poolId, uint256 amount, bytes32[] calldata merkleProof)
        external
        returns (uint256)
    { //@> Extra refunds aren't transferred even if they exist
        BiddingProject storage biddingProject = biddingProjects[projectId];
        require(biddingProject.claimDeadline > block.timestamp, Deadline_Has_Passed());

        Bid storage bid = biddingProject.bids[msg.sender];
        require(bid.amount > 0, No_Bid_Found());

        // Verify merkle proof
        bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId));

        require(!claimedNFT[leaf], Already_Claimed());
        require(MerkleProof.verify(merkleProof, biddingProject.vestingPools[poolId].merkleRoot, leaf), Invalid_Merkle_Proof());

        claimedNFT[leaf] = true;

        uint256 nftId = nftContract.mint(msg.sender);
        biddingProject.allocations[nftId].amounts.push(amount);
        biddingProject.allocations[nftId].vestingPeriods.push(bid.vestingPeriod);
        biddingProject.allocations[nftId].vestingStartTimes.push(biddingProject.endTime);
        biddingProject.allocations[nftId].claimedSeconds.push(0);
        biddingProject.allocations[nftId].claimedFlows.push(false);
        biddingProject.allocations[nftId].assignedPoolId = poolId;
        biddingProject.allocations[nftId].token = biddingProject.token;
        NFTBelongsToBiddingProject[nftId] = true;
        allocationOf[nftId] = biddingProject.allocations[nftId];
        emit NFTClaimed(projectId, msg.sender, nftId, poolId, amount);

        return nftId;
    }
```

### Internal Pre-conditions

1. Owner needs to call `createPool()` to set `hasExtraRefund` to `true` for at least one pool
2. Owner needs to call `finalizeBids()` to close the bidding project and set merkle roots

### External Pre-conditions

None

### Attack Path

1. Owner creates a pool with `hasExtraRefund = true`, signaling to users that excess funds will be refunded
2. Users place bids expecting partial refunds if they overbid relative to their final allocation
3. Owner finalizes bids and users claim their NFT allocations via `claimNFT()`
4. No refund logic executes because `hasExtraRefund` is never checked in `claimNFT`
5. After claim deadline passes, owner calls `withdrawPostDeadlineProfit()` and collects all remaining stablecoin, including amounts that should have been refunded to users

### Impact

Winning bidders in pools with `hasExtraRefund = true` suffer a loss of their excess stablecoin (the difference between their bid amount and what they should have paid based on final allocation). The protocol owner gains these excess funds that should have been returned to users. The exact loss amount depends on the difference between bid amounts and final allocations, but could be substantial in oversubscribed pools.

### PoC

The following test was ran and produced these logs: 
```bash
Ran 1 test for test/PoC.t.sol:DeadCodePoC
[PASS] testExtraRefundIsIgnored() (gas: 972606)
Logs:
  npm warn exec The following package was not found and will be installed: @openzeppelin/upgrades-core@1.44.2

  Bid Amount:      1000000000000000000000
  Allocation Cost: 800000000000000000000
  User Balance Before Claim: 1000000000000000000000
  User Balance After Claim:  1000000000000000000000
  Confirmed: User received 0 refund despite hasExtraRefund=true

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.74s (8.70ms CPU time)
```
The test:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";
import "forge-std/console.sol";

contract DeadCodePoC is Test {
    AlignerzVesting public vesting;
    AlignerzNFT public nft;
    Aligners26 public token;
    MockUSD public usdt;
    
    address public user;
    uint256 constant PROJECT_ID = 0;

    function setUp() public {
        user = makeAddr("user");
        token = new Aligners26("26Aligners", "A26Z");
        usdt = new MockUSD();
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");
        
        address payable proxy = payable(Upgrades.deployUUPSProxy(
            "AlignerzVesting.sol",
            abi.encodeCall(AlignerzVesting.initialize, (address(nft)))
        ));
        vesting = AlignerzVesting(proxy);
        
        nft.addMinter(address(vesting));
        
        // Setup User funds (USDT for bidding)
        usdt.mint(user, 2000 ether);
        vm.prank(user);
        usdt.approve(address(vesting), type(uint256).max);

        
        // The owner (this contract) needs Project Tokens to create a pool.
        
        token.approve(address(vesting), type(uint256).max);
        
    }
    
    function testExtraRefundIsIgnored() public {
        // 1. Launch Project
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp + 1 days,
            block.timestamp + 7 days,
            bytes32(0),
            false
        );

        // 2. Create Pool with `hasExtraRefund` set to TRUE
        // This transfers 10,000 tokens from Owner -> Vesting (requires approval above)
        vesting.createPool(PROJECT_ID, 10000 ether, 1 ether, true); 

        // Warp to bidding time
        vm.warp(block.timestamp + 1 days + 1);

        // 3. User Bids 1000 USDC
        uint256 bidAmount = 1000 ether;
        vm.prank(user);
        vesting.placeBid(PROJECT_ID, bidAmount, 30 days);

        // Warp to end
        vm.warp(block.timestamp + 7 days + 1);

        // 4. Finalize Bids
        // We act as backend. We decide the user actually only 'spent' 800 USDC
        // for their allocation. They should be refunded 200 USDC.
        uint256 allocationCost = 800 ether;
        
        // Leaf = Hash(user, amount, projectId, poolId)
        bytes32 leaf = keccak256(abi.encodePacked(user, allocationCost, PROJECT_ID, uint256(0)));
        
        bytes32[] memory roots = new bytes32[](1);
        roots[0] = leaf;
        
        vesting.finalizeBids(PROJECT_ID, bytes32(0), roots, 30 days);

        // 5. Claim NFT
        vm.startPrank(user);
        
        uint256 balanceBefore = usdt.balanceOf(user);
        
        // Call claimNFT with the allocation cost (800)
        vesting.claimNFT(PROJECT_ID, 0, allocationCost, new bytes32[](0));
        
        uint256 balanceAfter = usdt.balanceOf(user);
        
        vm.stopPrank();

        // 6. Verify Refund Missing
        // Expected Refund = Bid (1000) - Cost (800) = 200.
        uint256 expectedRefund = bidAmount - allocationCost;

        console.log("Bid Amount:     ", bidAmount);
        console.log("Allocation Cost:", allocationCost);
        console.log("User Balance Before Claim:", balanceBefore);
        console.log("User Balance After Claim: ", balanceAfter);
        
        // The test passes if the balance did NOT change (proving the refund logic is missing/dead code)
        assertEq(balanceAfter, balanceBefore, "Refund was received unexpectedly (Bug not present)");
        assertEq(expectedRefund, 200 ether, "Math check");
        
        console.log("Confirmed: User received 0 refund despite hasExtraRefund=true");
    }
}
```

### Mitigation

* Implement the refund logic: Add checks in `claimNFT()` to read `hasExtraRefund` and calculate/transfer excess stablecoin back to users based on their actual allocation vs. bid amount
  