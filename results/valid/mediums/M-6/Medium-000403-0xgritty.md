# [000403] Blacklisted users can update Bid and vestingPeriod
  
  ### Summary

- There is no check inside `AlignerzVesting::updateBid` to see if `msg.sender` is whitelisted.
- This allows blacklisted users to update bid and change vestingPeriod.


### Root Cause

- Lack of `isWhitelisted[msg.sender][projectId]` check inside `updateBid`. 


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

- User calls `AlignerzVesting::placeBid` to place bid on a particular project.
- Protocol identifies this user as a malicious one, so they blacklist him.
- This malicious user can still call `AlignerzVesting::updateBid` to change bid amount and vestingPeriod.


### Impact

- Blacklisted user can update their bid and vestingPeriod.


### PoC

_No response_

### Mitigation

_No response_
  