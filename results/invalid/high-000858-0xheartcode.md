# [000858] Cross-Project Token Drain Through ProjectId Parameter Manipulation
  
  ## Summary

Critical vulnerability in `claimTokens`, `splitTVS`, and `mergeTVS` functions allows users to drain tokens from incorrect projects by manipulating the `projectId` parameter without validation, enabling direct fund theft across project boundaries.

## Vulnerability Detail

The functions `claimTokens`, `splitTVS`, and `mergeTVS` accept a `projectId` parameter but **do not validate** that this projectId actually corresponds to the project that the NFT belongs to. This allows attackers to drain tokens from any project by providing wrong project IDs.

**Location**: [`AlignerzVesting.sol:941-946`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L941-L946)

```solidity
function claimTokens(uint256 projectId, uint256 nftId) external {
    address nftOwner = nftContract.extOwnerOf(nftId);
    require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());
    bool isBiddingProject = NFTBelongsToBiddingProject[nftId];
    (Allocation storage allocation, IERC20 token) = isBiddingProject ? 
    (biddingProjects[projectId].allocations[nftId], biddingProjects[projectId].token) : 
    (rewardProjects[projectId].allocations[nftId], rewardProjects[projectId].token);
    // CRITICAL: No validation that projectId matches the NFT's actual project!
}
```

### Root Cause Analysis

**Critical Gap**: All three vulnerable functions use the pattern:
1. User provides `projectId` parameter
2. Function loads allocation from `projects[projectId].allocations[nftId]` 
3. **Missing**: Validation that NFT actually belongs to that project
4. Function transfers tokens using wrong project's token contract

**Affected Functions**:
- [`claimTokens()`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L941) - Lines 945-947
- [`splitTVS()`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1055) - Lines 1062-1064  
- [`mergeTVS()`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002) - Lines 1007-1009

### Attack Mechanism

1. **Setup**: Attacker obtains NFT from Project A (with Token A)
2. **Manipulation**: Calls `claimTokens(projectB_id, nftA_id)` 
3. **Exploitation**: Function loads Project B's allocation and token contract
4. **Token Drain**: Transfers tokens from Project B's reserves to attacker

**Example Attack**:
```solidity
// Attacker owns NFT #50 from Project A (TokenX)
// Calls claimTokens with Project B's ID (TokenY)
claimTokens(projectB_id, 50);

// Function executes:
// 1. Validates attacker owns NFT #50 ✓  
// 2. Loads allocation from projectB.allocations[50] (wrong project!)
// 3. Uses TokenY.safeTransfer() instead of TokenX.safeTransfer()
// 4. Drains Project B's TokenY reserves
```

## Impact

Direct fund loss through cross-project exploitation
Users can drain tokens from any project by providing wrong projectId
 Projects can be completely drained of their token reserves  
Attackers can target high-value projects using low-value NFTs

### Exploitation Scenarios

**Scenario 1: High-Value Project Targeting**
- Attacker mints cheap NFT from low-value Project A
- Uses NFT to claim expensive tokens from Project B via wrong projectId
- Project B loses valuable tokens while Project A is unaffected

**Scenario 2: Empty Allocation Abuse**  
- Attacker exploits projects with sparse NFT allocations
- Accesses uninitialized allocation slots (default zero values)
- Causes accounting inconsistencies and state corruption

## Code Snippet

```solidity
// AlignerzVesting.sol:941-947 - claimTokens vulnerability
function claimTokens(uint256 projectId, uint256 nftId) external {
    address nftOwner = nftContract.extOwnerOf(nftId);
    require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());
    bool isBiddingProject = NFTBelongsToBiddingProject[nftId];
    (Allocation storage allocation, IERC20 token) = isBiddingProject ? 
    (biddingProjects[projectId].allocations[nftId], biddingProjects[projectId].token) : 
    (rewardProjects[projectId].allocations[nftId], rewardProjects[projectId].token);
    // Uses user-provided projectId without validation
}
```

**Similar patterns in**:
- [`splitTVS()`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1055-L1064)
- [`mergeTVS()`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002-L1009)

## Tool used

Manual Review

## Recommendation

Add project validation to all affected functions:

```solidity
function claimTokens(uint256 projectId, uint256 nftId) external {
    address nftOwner = nftContract.extOwnerOf(nftId);
    require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());
    
    // FIX: Validate projectId matches NFT's actual project
    bool isBiddingProject = NFTBelongsToBiddingProject[nftId];
    Allocation memory nftAllocation = allocationOf[nftId];
    
    if (isBiddingProject) {
        require(biddingProjects[projectId].token == nftAllocation.token, "Invalid project ID");
    } else {
        require(rewardProjects[projectId].token == nftAllocation.token, "Invalid project ID");
    }
    
    // Continue with existing logic...
}
```
  