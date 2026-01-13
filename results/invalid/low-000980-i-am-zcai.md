# [000980] Users will be denied legitimate claims exactly at claimDeadline [Low]
  
  
### Summary

The exclusive deadline comparisons in multiple claim functions will cause legitimate claims to be denied for users as they attempt to claim exactly when `block.timestamp` equals `claimDeadline`, despite documentation indicating claims should be permitted until the deadline.

### Root cause

In `protocol/src/contracts/vesting/AlignerzVesting.sol#L508` the deadline check uses strict inequality (`<`) rather than inclusive comparison (`<=`), creating a boundary condition where claims are rejected at the exact deadline timestamp.

### Internal pre-conditions

1. Admin needs to set `claimDeadline` to be exactly equal to the current block timestamp when a user attempts to claim

### External pre-conditions

None required.

### Attack path

1. User calls `claimRewardTVS()` when `block.timestamp` equals `rewardProject.claimDeadline`
2. The function reverts with `Deadline_Has_Passed()` error despite the deadline not having technically passed
3. Similar rejections occur for `claimStablecoinAllocation()`, `claimRefund()`, and `claimNFT()` functions

### Impact

The users suffer denied access to legitimate claims during the final moment of the claim window. Users cannot claim their allocations when transactions are mined at blocks with timestamps exactly matching the deadline, potentially causing confusion given the documented behavior implies inclusive deadline semantics.

### Mitigation

Consider updating the deadline comparison logic to use inclusive bounds that align with the documented semantics. One approach would be to modify the strict inequalities to allow claims at the exact deadline timestamp:

```diff
- require(block.timestamp < rewardProject.claimDeadline, Deadline_Has_Passed());
+ require(block.timestamp <= rewardProject.claimDeadline, Deadline_Has_Passed());
```

```diff
- require(biddingProject.claimDeadline > block.timestamp, Deadline_Has_Passed());
+ require(biddingProject.claimDeadline >= block.timestamp, Deadline_Has_Passed());
```
  