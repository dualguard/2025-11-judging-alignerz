# [000405] Compunding Rounding Loss in `splitTVS` Flow Allocation Computation
  
  ### Summary

The [_computeSplitArrays](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113-L1141) function in `AlignerzVesting.sol` calculates split allocation amounts using integer division with a percentage basis: `alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT`. When splitting across multiple recipients, rounding down on each split can leave dust amounts unallocated. The final split should receive the remainder to preserve the exact total allocation, but instead it also uses the same truncating formula, potentially losing wei-level precision. This is further compounded multiple times depending on `nbOfFlows` & `nbOfTVS`

### Root Cause

The function applies the formula `(baseAmounts[j] * percentage) / BASIS_POINT` uniformly to all flow indices in a loop, including the final split. Integer division truncates fractional wei amounts, and these truncation losses are not recovered or assigned to any recipient. The loop does not track cumulative allocated amounts or adjust the final allocation to account for rounding errors.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path


Consider `baseAmounts[0] = 99`, splitting `[5000, 5000]`: equal
- Split 1: `99 * 5000 / 10000 = 49` (truncated from 49.5)
- Split 2: `99 * 5000 / 10000 = 49` (truncated from 49.5)
- Total allocated: `49 + 49 = 98`
- **Lost**: `1` unit due to rounding 

### Impact

## Impact
- **Precision loss**: Small amounts of tokens lost per split due to truncating division and this loss occurs multiple times depending on `nbOfFlows` & `nbOfTVS`

### PoC

_No response_

### Mitigation

 Modify the split logic to process all flows except the last one using the standard percentage calculation formula. For the last flow index, instead of applying the percentage formula (which causes rounding), directly calculate it as: `original amount - sum of all previously allocated amounts`. This ensures that all rounding remainders accumulate in the final flow, guaranteeing zero dust loss and perfect conservation of the total allocation amount.

  