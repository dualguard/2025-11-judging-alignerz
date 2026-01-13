# [000942] `splitTVS` is completely unusable due to uninitialized memory arrays causing an Out-Of-Bounds revert
  
  ### Summary

# H-1
Uninitialized memory arrays in `AlignerzVesting::_computeSplitArrays()` will cause a denial of service for all users attempting to split their TVS NFTs as any user calling `splitTVS()` will encounter an array out-of-bounds panic (0x32) that makes the feature completely unusable

### Root Cause

In [_computeSplitArrays](https://github.com/dualguard/2025-11-alignerz/blob/ed2b9764732b5cdc4b8b35209cd2263f72210f31/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113-L1141), the function  declares an `Allocation memory alloc struct` but never initializes its array fields (amounts, vestingPeriods, vestingStartTimes, claimedSeconds, claimedFlows) before attempting to assign values to array indices. In Solidity, memory arrays must be explicitly sized using new uint256[](size) before accessing indices. Without initialization, accessing `alloc.amounts[j]` at line 1132 triggers panic code 0x32 (array out-of-bounds access).

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

- User calls `splitTVS()` with valid parameters (project ID, percentages array, NFT ID) AlignerzVesting.sol:1054-1060
Function validates ownership and retrieves the allocation 
At line L1088,` _computeSplitArrays()` is called to compute the split allocation 
- Inside _computeSplitArrays(), the loop attempts to access alloc.amounts[j] without prior initialization 
- Transaction reverts with panic 0x32 before any split logic executes

### Impact

Users cannot split their TVS NFTs into multiple allocations, rendering the entire split functionality unusable. 

### PoC

```solidity
 function test_SplitTVS_Reverts_DueTo_MemoryBug() public {
        // 1. SETUP: Launch Project & Pool
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1000, "0x0", true);
        vesting.createPool(PROJECT_ID, 100_000 ether, 0.01 ether, false);
        
        // Whitelist our test bidder AND a dummy bidder
        address bidder = bidders[0];
        address dummyBidder = bidders[1];
        address[] memory whitelist = new address[](2);
        whitelist[0] = bidder;
        whitelist[1] = dummyBidder;
        vesting.addUsersToWhitelist(whitelist, PROJECT_ID);
        vm.stopPrank();

        // 2. PLACE BIDS (Main Bidder + Dummy Bidder)
        // Main Bidder
        vm.startPrank(bidder);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 100); 
        vm.stopPrank();

        // Dummy Bidder (Required to make Merkle Tree > 1 leaf)
        vm.startPrank(dummyBidder);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 100);
        vm.stopPrank();

        // 3. FINALIZE (Prepare Merkle Proofs with 2 leaves)
        BidInfo[] memory bids = new BidInfo[](2);
        bids[0] = BidInfo({
            bidder: bidder,
            amount: BIDDER_USD,
            vestingPeriod: 100,
            poolId: 0,
            accepted: true
        });
        bids[1] = BidInfo({
            bidder: dummyBidder,
            amount: BIDDER_USD,
            vestingPeriod: 100,
            poolId: 0,
            accepted: true
        });

        // Generate root and store proofs
        bytes32 root = generateMerkleProofs(bids, 0);
        bytes32[] memory roots = new bytes32[](1);
        roots[0] = root;

        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), roots, 1 days);

        // 4. CLAIM NFT
        vm.startPrank(bidder);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[bidder]);
        
        // 5. EXECUTE THE BUG
        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5000; 
        percentages[1] = 5000;

        // *** THE CONFIRMATION ***
        // We expect Panic(0x32) which is Array Out of Bounds
        vm.expectRevert(); 
        vesting.splitTVS(PROJECT_ID, percentages, nftId);
        
        vm.stopPrank();
    }
```
Place this in the  AlignerzVestingProtocolTest 

### Mitigation

_No response_
  