# [000618] Stable coins can’t be claimed at the exact end time of claimDeadline
  
  ### Summary

KOLs can’t claim their stable coins at the exact claimDeadline. They can claim before as intended, and after (due to an access control issue I submit in another report) the claimDeadline, but at the exact moment of claimDeadline, they can’t claim.

Note: The same issue exists for rewards claim.

### Details

KOLs call `claimStablecoinAllocation` to claim their rewards, they have to follow this check there:

```jsx
        require(block.timestamp < rewardProject.claimDeadline, Deadline_Has_Passed());

```

They can also call `distributeStablecoinAllocation` to claim their rewards after the deadline. They have to follow this check there now:

```jsx
        require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());

```

Now, the first clause requires that block.timestamp is less than claimDeadline, and the latter requires that it is after. At the exact time that block.timestamp == claimDeadline, both functions cannot be called, hence stable coin allocation cannot be claimed.

I submit this issue as low severity, for a brief moment, there is a denial of service.
  