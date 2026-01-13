# [000844] Cross-Project Merge Attack Allows NFT Allocation Manipulation
  
  ## Summary

The `mergeTVS` function allows users to merge NFT allocations using arbitrary project mappings by manipulating the `projectIds` parameter, enabling attackers to reference wrong project data during merge operations and potentially corrupt allocation state.

## Vulnerability Detail

The `mergeTVS` function accepts user-provided `projectIds` array and uses these values to look up NFT allocations, but doesn't validate that each NFT actually belongs to the specified project. This allows attackers to reference NFTs using incorrect project mappings.

**Location**: [`AlignerzVesting.sol:1002`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002) and [`_merge() function`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1028-L1034)

```solidity
function mergeTVS(uint256 projectId, uint256 mergedNftId, uint256[] calldata projectIds, uint256[] calldata nftIds) external {
    // ... validation ...
    for (uint256 i; i < nbOfNFTs; i++) {
        feeAmount += _merge(mergedTVS, projectIds[i], nftIds[i], token);
        //                           ^^^^^^^^^^^^^^ User-provided project ID
    }
}

function _merge(Allocation storage mergedTVS, uint256 projectId, uint256 nftId, IERC20 token) internal {
    require(msg.sender == nftContract.extOwnerOf(nftId), Caller_Should_Own_The_NFT());
    
    bool isBiddingProjectTVSToMerge = NFTBelongsToBiddingProject[nftId];
    (Allocation storage TVSToMerge, IERC20 tokenToMerge) = isBiddingProjectTVSToMerge ?
    (biddingProjects[projectId].allocations[nftId], biddingProjects[projectId].token) :
    //               ^^^^^^^^^ BUG: Uses attacker's projectId, not NFT's actual project
    (rewardProjects[projectId].allocations[nftId], rewardProjects[projectId].token);
}
```

### Root Cause Analysis

**Critical Gap**: The function flow allows project boundary violations:
1. User provides `projectIds[]` array for NFTs to merge
2. `_merge()` uses `projectId` to look up allocation: `projects[projectId].allocations[nftId]`
3. **Missing**: Validation that NFT actually belongs to that project
4. Function merges data from wrong project mapping

**Ownership Validation Present**: The function does validate NFT ownership (`require(msg.sender == nftContract.extOwnerOf(nftId))`), which limits but doesn't eliminate the attack surface.

### Attack Mechanism

**Scenario**: Cross-project allocation manipulation
1. **Attacker Setup**: Owns NFT #100 originally from Project A
2. **Attack Call**: `mergeTVS(projectA, myNftId, [projectB], [100])`
3. **Wrong Lookup**: Function retrieves `biddingProjects[projectB].allocations[100]` instead of `biddingProjects[projectA].allocations[100]`
4. **State Corruption**: Merges empty/uninitialized allocation data or wrong project's allocation

**Impact Scenarios**:
- **Empty Allocation Merge**: If Project B has no allocation for NFT #100, merges empty/default values
- **Cross-Project Data**: If Project B has different allocation for NFT #100, merges unintended data
- **Accounting Inconsistency**: Creates mismatched state between actual NFT project and merged data

## Impact

**High Severity**: Project boundary violation enabling allocation manipulation

- **NFT Allocation Manipulation**: Attackers can merge allocations from wrong project mappings
- **Empty Allocation Abuse**: Attackers can merge empty/default allocations to manipulate existing allocations
- **Cross-Project State Corruption**: Merge operations can corrupt allocation data using invalid project references
- **Protocol Data Integrity**: Breaks assumption that NFTs only interact with their assigned projects

### Exploitation Examples

**Example 1: Empty Allocation Dilution**
```solidity
// User owns NFT #50 from Project A with 1000 tokens
// Project B has empty allocation for NFT #50
// Attack: mergeTVS(projectA, myNft, [projectB], [50])
// Result: Merges empty allocation, potentially reducing combined allocation
```

**Example 2: Cross-Project Data Access**
```solidity
// Project A NFT #100: 500 tokens, 90-day vesting
// Project B NFT #100: 200 tokens, 30-day vesting (different user's NFT)
// Attack: mergeTVS(projectA, myNft, [projectB], [100]) 
// Result: Merges wrong project's allocation data
```

## Code Snippet

```solidity
// AlignerzVesting.sol:1020-1022 - User controls project references
for (uint256 i; i < nbOfNFTs; i++) {
    feeAmount += _merge(mergedTVS, projectIds[i], nftIds[i], token);
    //                           ^^^^^^^^^^^^^^ User-provided project ID
}

// AlignerzVesting.sol:1032-1034 - Uses attacker-controlled projectId  
(Allocation storage TVSToMerge, IERC20 tokenToMerge) = isBiddingProjectTVSToMerge ?
(biddingProjects[projectId].allocations[nftId], biddingProjects[projectId].token) :
//               ^^^^^^^^^ BUG: Uses attacker's projectId instead of NFT's actual project
(rewardProjects[projectId].allocations[nftId], rewardProjects[projectId].token);
```

**Ownership Check Present** (limits attack scope):
```solidity
require(msg.sender == nftContract.extOwnerOf(nftId), Caller_Should_Own_The_NFT());
```
  