# [000607] claim is blocked even if the deadline has not passed
  
  ### Summary



Using < instead of <= when checking the claim deadline in claimRewardTVS will cause a denial of reward claims for users of the reward project as users will be unable to claim their rewards exactly at the claim deadline.

### Root Cause

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L506


The claimRewardTVS function currently checks the claim deadline using the line below 

```javascript
 function claimRewardTVS(uint256 rewardProjectId) external {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        require(block.timestamp < rewardProject.claimDeadline, Deadline_Has_Passed()); // @audit claim is blocked even if the deadline has not passed
        address kol = msg.sender;
        _claimRewardTVS(rewardProjectId, kol);
    }
```
As you can see 
```javascript
 require(block.timestamp < rewardProject.claimDeadline, Deadline_Has_Passed()); 
```

This prevents users from claiming rewards exactly at the claim deadline `(block.timestamp == rewardProject.claimDeadline)`.As a result, users are blocked from claiming rewards even though the intended claim period should include the last second.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path


1 User waits until the final moment of the claim window, intending to claim their reward NFT or vested tokens at the exact claim deadline.

2 block.timestamp reaches exactly rewardProject.claimDeadline, which should logically still be part of the allowed claim period.

The user calls:

`claimRewardTVS(rewardProjectId)`


3 The function executes the check:

`require(block.timestamp < rewardProject.claimDeadline, "Deadline_Has_Passed()");`


4 Because `block.timestamp == rewardProject.claimDeadline`, the condition evaluates to false.

5 The transaction reverts, preventing the user from claiming their allocation despite the deadline not having passed.

6 The user has permanently lost the ability to claim, because every subsequent block has a timestamp greater than the deadline.


### Impact

Users cannot claim rewards on the exact last block of the claim window.

### PoC

_No response_

### Mitigation


Use a `<=` comparison to allow claiming up to and including the deadline:

`require(block.timestamp <= rewardProject.claimDeadline, "Deadline_Has_Passed()");`

This ensures the entire intended claim period is accessible to users.
  