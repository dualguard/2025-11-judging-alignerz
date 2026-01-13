# [000151] setTVSAllocation doesn't enforce the divisor
  
  If an admin changes the vestingPeriodDivisor after a user has already placed a bid, that user can be prevented from updating their bid. The user is forced to also change their vesting period to a new, valid multiple just to update their bid amount, even if they didn't want to. 

```
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        rewardProject.vestingPeriod = vestingPeriod;
```

Add the same validation used elsewhere, e.g. require(vestingPeriod > 0 && (vestingPeriod == 1 || vestingPeriod % vestingPeriodDivisor == 0), Vesting_Period_Is_Not_Multiple_Of_The_Base_Value()); before assigning rewardProject.vestingPeriod.
  