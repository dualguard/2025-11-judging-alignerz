# [000023] `claimDividends` reverts due to time reconfiguration
  
  ### Summary

The contract stores `claimedSeconds` cumulatively under the assumption that `secondsPassed` is always increasing and tied to a fixed `startTime`. However, if a user has already made a claim, so `claimedSeconds` reflects the previous `secondsPassed` , and later the owner updates startTime to a higher value, the next calculation `secondsPassed = block.timestamp - newStartTime` may become smaller than the stored `claimedSeconds`. This leads to a subtraction where (secondsPassed - claimedSeconds) underflows, causing a revert.

### Root Cause

https://github.com/dualguard/2025-11-alignerz-0xAlix2/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L198

`claimDividends` assumes `startTime` and `vestingPeriod` are immutable and that `secondsPassed` is always ≥ `claimedSeconds`, but the owner can arbitrarily modify both values.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Users make initial claims, making their claimedSeconds non-zero.

2. The owner updates startTime to a later value, reducing the computed secondsPassed = block.timestamp - startTime.

3. On the next claim, secondsPassed - claimedSeconds underflows, causing a revert.

### Impact

Users who have already claimed become unable to claim again if the owner updates startTime forward. Since claimedSeconds reflects the old schedule while the new secondsPassed becomes smaller, the subtraction underflows and subsequent claimDividends calls revert for those users.

### PoC

_No response_

### Mitigation

Disallow changing startTime and vestingPeriod once dividends have been set / once any user has claimed.
  