# [000934] Strict < Check Creates Claim Failure at Exact Deadline
  
  ### Summary

Using a strict < check at lines [508](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L508) and [517](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L517) instead of <= will cause a denial-of-service at the exact deadline timestamp for KOL users, as user claims will revert, while other parts of the contract 
use > checks—treating <= claimDeadline as “deadline not passed.” This inconsistent deadline logic creates a 
1-timestamp gap where user claims incorrectly fail.


### Root Cause


At [vesting.sol:508](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L508) and v[esting.sol:517](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L517), the contract uses:

`require(block.timestamp < rewardProject.claimDeadline, Deadline_Has_Passed());`


but in other functions (e.g., distributeRewardTVS, distributeStablecoinAllocation, withdrawPostDeadlineProfit), the contract uses:

`require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());`


This shows the contract implicitly treats <= claimDeadline as valid pre-deadline, but the user-claim functions incorrectly reject the equality case due to <.


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

User submits a claim close to the deadline.

The block containing the transaction has **block.timestamp == claimDeadline**.

Lines 508 and 517 check **block.timestamp < claimDeadline → fails**.

Meanwhile, other functions imply that <= claimDeadline should be valid, creating inconsistent behavior.

The user’s claim reverts even though the deadline has not technically passed based on the rest of the system.


### Impact

- KOL users cannot claim tokens when their transaction lands exactly at claimDeadline.

- trust damage: users see unexpected reverts at the deadline.

### PoC

_No response_

### Mitigation

Update both lines:

`require(block.timestamp <= rewardProject.claimDeadline, Deadline_Has_Passed());`


This makes user-claim logic consistent with the rest of the contract’s deadline semantics 
(<=  deadline not passed, > deadline passed).
  