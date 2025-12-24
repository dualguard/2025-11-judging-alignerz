# [000443] No refund or profit withdrawal possible at `block.timestamp == biddingProject.claimDeadline`
  
  ### Summary

The vesting contract allows users to claim a refund for unaccepted bids using `claimRefund` before the `biddingProject.claimDeadline`, and for the protocol to withdraw the refund after the `biddingProject.claimDeadline`.

However, this is not possible for both parties to call any of these function when `block.timestamp == biddingProject.claimDeadline`. Funds will be locked for that small period of time.

### Root Cause

`claimRefund`'s first require should include the `claimDeadline` in the condition.

### Impact

Funds locked for a small period of time.

### Mitigation

```diff
-   require(biddingProject.claimDeadline > block.timestamp, Deadline_Has_Passed());
+   require(biddingProject.claimDeadline >= block.timestamp, Deadline_Has_Passed());
```
  