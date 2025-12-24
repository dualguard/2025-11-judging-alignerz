# [000078] Users will corrupt allocation data by merging BiddingProject and RewardProject NFTs
  
  ### Summary

The lack of project type validation in `_merge()` at line 1031 will cause data corruption and incorrect vesting calculations for NFT holders as users will merge NFTs from different project types (BiddingProject and RewardProject) into a single allocation.

### Root Cause

In [AlignerzVesting.sol:1031](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1031) the `_merge()` function determines the project type of the NFT being merged (`isBiddingProjectTVSToMerge`) but does not validate that it matches the project type of the target merged NFT (`isBiddingProject` from the parent `mergeTVS()` function). 

### Internal Pre-conditions

1. User needs to own at least one `BiddingProject` NFT and one `RewardProject` NFT
2. Both NFTs need to have the same underlying token address (to pass the `Different_Tokens()` check at line 1035)

### External Pre-conditions

None

### Attack Path

1. User calls `mergeTVS()` with a BiddingProject NFT as mergedNftId and includes a RewardProject NFT in the nftIds array (or vice versa)
2. The function retrieves the merged NFT's allocation from `biddingProjects[projectId].allocations[mergedNftId]` 
3. `_merge()` is called for each NFT to merge, and retrieves the NFT's allocation from `rewardProjects[projectId].allocations[nftId]` 
4. The function only checks that tokens match (line 1035) but does not verify project type compatibility
5. RewardProject allocation data (with different vesting start times, periods, and claiming logic) gets pushed into the BiddingProject allocation arrays 
6. The merged allocation now contains mixed data from incompatible project types

### Impact

Users suffer data corruption in their merged NFT allocations. The merged allocation will contain:

- Mixed vesting start times (BiddingProject uses `endTime`, RewardProject uses `startTime`)
- Inconsistent `assignedPoolId` values (BiddingProject has actual pool IDs, RewardProject always uses 0)
- Incorrect claiming calculations when `claimTokens()` is called, as it cannot distinguish between the different project types within a single allocation

This breaks the protocol's project type separation model and can lead to incorrect token distribution, early claiming, or inability to claim tokens at all.

### PoC

N/A

### Mitigation

Add a project type validation check in `_merge()`:

```diff
function _merge(Allocation storage mergedTVS, uint256 projectId, uint256 nftId, IERC20 token) internal returns (uint256 feeAmount) {  
    require(msg.sender == nftContract.extOwnerOf(nftId), Caller_Should_Own_The_NFT());  
      
    bool isBiddingProjectTVSToMerge = NFTBelongsToBiddingProject[nftId];  
      
+    // Add validation: both NFTs must be from the same project type  
+    bool isBiddingProjectMerged = NFTBelongsToBiddingProject[mergedNftId];  
+    require(isBiddingProjectTVSToMerge == isBiddingProjectMerged, "Cannot merge different project types");  
      
    (Allocation storage TVSToMerge, IERC20 tokenToMerge) = isBiddingProjectTVSToMerge ?  
        (biddingProjects[projectId].allocations[nftId], biddingProjects[projectId].token) :  
        (rewardProjects[projectId].allocations[nftId], rewardProjects[projectId].token);  
    require(address(token) == address(tokenToMerge), Different_Tokens());  
      
    // ... rest of function  
}
```
  