# [000090] Unbounded loop in withdrawAllPostDeadlineProfits will cause eventual DoS
  
  ### Summary

The unbounded loop in `withdrawAllPostDeadlineProfits` will cause eventual DoS for the owner as the protocol creates more bidding projects over time, since the function iterates through all projects ever created without any gas limit consideration.

### Root Cause

In [AlignerzVesting.sol:894-897](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L894-L897), the `withdrawAllPostDeadlineProfits()` iterates through all bidding projects using `biddingProjectCount` as the loop bound.  Since `biddingProjectCount` only increments and never decrements, the gas cost grows linearly with each new project created, eventually exceeding the block gas limit and making the function permanently unusable.

### Internal Pre-conditions

1. Protocol needs to create multiple bidding projects over time through `createBiddingProject()`
2. `biddingProjectCount` needs to grow to a sufficiently large number where the loop exceeds block gas limits

### External Pre-conditions

N/A

### Attack Path

1. Protocol operates normally creating bidding projects over time
2. `biddingProjectCount` increases with each new project (e.g., reaches 100, 200, 500 projects)
3. Owner calls `withdrawAllPostDeadlineProfits()` to withdraw profits from all projects
4. Transaction reverts due to out-of-gas error as the loop iterates through hundreds of projects
5. Function becomes permanently unusable once project count exceeds gas limit threshold

### Impact

The owner cannot user the convience fucntion `withdrawAllPostDeadlineProfits()` anymore to withdraw in a single transaction once the protocol scales. However, no funds are lost or locked because alternative withdrawal methods exist: `withdrawPostDeadlineProfit(projectId)` for single projects and `withdrawPostDeadlineProfits(uint256[] calldata projectIds)` for batch withdrawals. 

### PoC

N/A 

### Mitigation

There is two possible options to solve this problem:

Option1: Depercate `withdrawAllPostDeadlineProfits()` and document that owner should user the batch withdrawal function instead `withdrawPostDeadlineProfits()`

Option2: Add a startIndex and endIndex to limit iteration per call. This option is better for the owner because it easier to call from calling batch of project ids. 

```diff
function withdrawAllPostDeadlineProfits(uint256 startIndex, uint256 endIndex) external onlyOwner {  
    require(endIndex <= biddingProjectCount, Invalid_Project_Id());  
    require(startIndex < endIndex, Invalid_Range());  
      
    for (uint256 i = startIndex; i < endIndex; i++) {  
        _withdrawPostDeadlineProfit(i);  
    }  
}
```

  