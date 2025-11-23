# [000488] Users can mergeTVS() with different assignedPoolId
  
  ### Summary

While merging NFTs using `mergeTVS()`, users can use `mergedNftId` where `assignedPoolId` are different. Later, when the users claim their tokens. The emitted event with emit wrong `allocation.assignedPoolId`.

```solidity
emit TokensClaimed(projectId, isBiddingProject, allocation.assignedPoolId, allocation.isClaimed, nftId, allClaimableSeconds, block.timestamp, msg.sender, amountsClaimed);
```

### Root Cause

After merging, only the `assignedPoolId` of `mergedNftId` is kept. So, whenever user claim their tokens, the event will always emit `assignedPoolId`. Even if the original NFT had different `assignedPoolId`.

- https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1039-L1044
- https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L974

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

Whenever a user merge 2 NFTs with different `assignedPoolId`, one 1 is kept. 

### Impact

Incorrect data emitted in `TokensClaimed()` event

### PoC

_No response_

### Mitigation

Convert `assignedPoolId` variable into an array to keep track of different PoolIds of different allocations.
```diff
    /// @notice Represents an allocation
    /// @dev Tracks the allocation status and vesting progress
    struct Allocation {
        uint256[] amounts; // Amount of tokens committed for this allocation for all flows
        uint256[] vestingPeriods; // Chosen vesting duration in seconds for all flows
        uint256[] vestingStartTimes; // start time of the vesting for all flows
        uint256[] claimedSeconds; // Number of seconds already claimed for all flows
        bool[] claimedFlows; // Whether flow is claimed
        bool isClaimed; // Whether TVS is fully claimed
        IERC20 token; // The TVS token
++      uint256[] assignedPoolId;
--      uint256 assignedPoolId; // Relevant for bidding projects: Id of the Pool (poolId=0; pool#1 / poolId=1; pool#2 /...) - for reward projects it will be 0 as default
    }

```
  