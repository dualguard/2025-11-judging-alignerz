# [000621] Users can frontrun `finalizeBids` and place a bid, breaking the assumption of `endTimeHash`
  
  ### Summary

In `launchBiddingProject` function, the `endTime` is going to be set to a much higher value than it actually is, so users won't know the real `endTime`, and the real `endTime` is going to be stored in `endTimeHash` as a hash.
But the user can still wait and frontrun `finalizeBids` function and bid on the right project


### Root Cause

users can front run [finalizebids](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L783C14-L783C26)

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. bid project has started, users won't know the end time of the bid, and it's intended 
2. owner will call finalizeBids function, but a user front run and place a bid
3. The purpose of not knowing the endTime is broken

### Impact

The purpose of having an `endTimeHash` is broken, as users can front-run finalizeBids and bid at the last moment

### PoC

_No response_

### Mitigation

_No response_
  