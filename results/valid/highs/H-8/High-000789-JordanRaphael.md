# [000789] Infinite loop in getUnclaimedAmounts due to skipped array increment
  
  ### Summary

A missing loop increment in `DividendDistributor.getUnclaimedAmounts` will cause a revert / denial of service for users interacting with the DividendDistributor as any actor calling this function with allocations that trigger the skipped increment will cause the function to enter an infinite loop and revert.


### Root Cause

In `A26ZDividendDistributor.sol:20-37` the `for` loop in `getUnclaimedAmounts` does not increment `i` when `claimedFlows[i]` is `true` or `claimedSeconds[i] == 0`. This causes the loop to repeatedly evaluate the same index and revert.

[skipping the increment](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L149-L152)

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Any user calls `getUnclaimedAmounts(nftId)` with an NFT that has allocations where `claimedFlows[i]` is `true` or `claimedSeconds[i] == 0`.
2. The loop does not increment `i` for those elements.
3. The function enters an infinite loop, consuming all gas and reverting the transaction.
4. As a result, the user cannot query their unclaimed amounts for that NFT.


### Impact

- Users cannot retrieve unclaimed dividend amounts, effectively denying access to their tokens until the contract is fixed.
- Any higher-level functions that depend on `getUnclaimedAmounts` may also fail, potentially blocking distributions.


### PoC

_No response_

### Mitigation

Always increment `i` on each iteration, even when skipping elements.

  