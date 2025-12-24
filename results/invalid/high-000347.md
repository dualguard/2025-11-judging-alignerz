# [000347] Fee deduction in `AlignerzVesting.sol::splitTVS` modifies original allocation before splitting causing double reduction and token loss
  
  ### Summary

The contract subtracts fees before calculating the split in `AlignerzVesting::splitTVS`, so the split is based on a reduced amount. This means users receive less than what they should because the split is calculated after the allocation has already been reduced.

### Root Cause

In `AlignerzVesting::splitTVS` at https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1070, 
the function executes `allocation.amounts = newAmounts` (where `newAmounts` has fees deducted) immediately after calling `calculateFeeAndNewAmountForOneTVS`, before the loop that computes splits via `_computeSplitArrays(allocation, percentage, nbOfFlows)`. Because of this, the split is based on the reduced post-fee amounts instead of the original allocation.

### Internal Pre-conditions

1. The user must own a TVS NFT with a non-zero allocation
2. The user must call `splitTVS` function with valid percentages that sum to 10000 (100%)
3. The allocation must have sufficient token balance
4. Split fee rate must be non-zero

### External Pre-conditions

This occurs in every split operation

### Attack Path

1. User has TVS with allocation.amounts = [400e18, 300e18, 300e18] (total: 1000e18 tokens)
2. User calls `splitTVS(projectId, [5000, 5000], tvsId)` to split 50/50
3. Contract calculates fees: `calculateFeeAndNewAmountForOneTVS(50, [400e18, 300e18, 300e18], 3)`
4. Due to accumulation bug, returns: feeAmount = 5e18, newAmounts = [398e18, 296.5e18, 295e18] (total: 989.5e18). NB: I'm assuming the bug here has not been fixed.
5. Contract immediately modifies original: `allocation.amounts = newAmounts` (now [398e18, 296.5e18, 295e18])
6. Contract transfers fee to treasury: `token.safeTransfer(treasury, 5e18)`
7. Contract then computes splits from the REDUCED allocation:
   - Split 1 (50%): 50% of [398e18, 296.5e18, 295e18] = [199e18, 148.25e18, 147.5e18] (total: 494.75e18)
   - Split 2 (50%): 50% of [398e18, 296.5e18, 295e18] = [199e18, 148.25e18, 147.5e18] (total: 494.75e18)
8. User receives total: 989.5e18 tokens across both splits
9. Treasury received: 5e18 tokens
10. Total accounted: 994.5e18 tokens
11. Missing: 5.5e18 tokens permanently lost

### Impact

Users suffer compounding losses from both the fee accumulation bug AND the premature fee deduction:

### PoC

_No response_

### Mitigation

_No response_
  