# [000930] Infinite Loop, Wrong Fee Logic, Missing Return Break calculateFeeAndNewAmountForOneTVS()
  
  ### Summary

Besides the memory allocation bug, [calculateFeeAndNewAmountForOneTVS() ](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-#L173) contains multiple severe logic errors: missing loop increment, incorrect cumulative fee subtraction, and absence of a return statement. These errors cause the function to either revert or behave incorrectly when executed. Even after fixing memory allocation, the function still cannot execute successfully, making all TVS merges and splits impossible.


### Root Cause



Problems:

1. Missing loop increment (i++)

- infinite loop → out-of-gas revert.

2. Incorrect fee logic

- subtracts cumulative fees from each flow instead of per-flow fee.

3. Missing return statement

- function cannot complete execution.

Even when memory allocation is fixed, the function remains broken.


### Internal Pre-conditions

1. Assuming memory allocation is correctly added, _(as specified in my another report)_ :

newAmounts = new uint256[](length);

2.  User calls mergeTVS() or splitTVS().

3. The vesting contract calls:

(feeAmount, newAmounts) = calculateFeeAndNewAmountForOneTVS(...)


### External Pre-conditions

_No response_

### Attack Path

Failure Path

1. Function starts loop.

2. i never increments - infinite loop.

3. Eventually out-of-gas - revert.

4. Or if increment is added later, cumulative fee subtraction produces incorrect vesting amounts.

5. Even with both fixed, missing return results in revert.


### Impact


* Merging and splitting can never succeed even if memory is allocated, the function remains logically incorrect.

* Breaks entire TVS restructuring design


### PoC

_No response_

### Mitigation


Fix all remaining issues:

1. Add loop increment
unchecked { ++i; }

2. Use per-flow fee logic
uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
feeAmount += fee;
newAmounts[i] = amounts[i] - fee;

3. Add return statement
return (feeAmount, newAmounts);
  