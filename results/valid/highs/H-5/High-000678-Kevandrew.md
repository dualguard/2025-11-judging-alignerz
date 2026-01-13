# [000678] [H-5] Merge/split fee helper subtracts cumulative fees, corrupting later flows
  
  ### Summary

`calculateFeeAndNewAmountForOneTVS` accumulates the running total fee (`feeAmount`) and subtracts that cumulative value from each flow’s amount, so the second flow is reduced by the sum of the first two fees, the third flow loses all three fees, etc. Even once Issue #3  's missing `++i` / allocation bugs are fixed, merge/split outputs are still wrong and can underflow.

### Root Cause

In `FeesManager.sol:169-173`, the loop does `feeAmount += calculateFeeAmount(feeRate, amounts[i]); newAmounts[i] = amounts[i] - feeAmount;`. The subtraction should use only the per-flow fee, but it subtracts the entire cumulative `feeAmount`, penalizing later flows multiple times and potentially causing reverts if `feeAmount > amounts[i]`.

### Internal Pre-conditions

  1. Merge or split functionality is enabled (array allocations / `++i` fixed so the function runs).
  2. There are at least two vesting flows in `amounts`.

### External Pre-conditions

None.

### Attack Path

  1. User calls `mergeTVS` or `splitTVS`.
  2. `calculateFeeAndNewAmountForOneTVS` iterates flows; after processing flow 0, `feeAmount` equals that flow’s fee.
  3. When processing flow 1, it adds the second fee to `feeAmount` and subtracts the sum of both fees from `amounts[1]`, overcharging the second flow.
  4. With more flows, the error compounds; flows near the end lose the sum of all previous fees, often causing underflow and reverting.

### Impact

Merge and split outputs produce incorrect vesting amounts, and transactions with multiple flows likely revert due to underflow. Users cannot rely on merge/split accounting; funds may be silently discarded or operations fail.

### PoC

```
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {FeesManager} from "../src/contracts/vesting/feesManager/FeesManager.sol";

contract FeeHelperCumulativeTest is Test {
    FeeHelperHarness private helper;

    function setUp() public {
        helper = new FeeHelperHarness();
    }

    function test_calculateFeeAndNewAmount_overchargesLaterFlows() public {
        uint256[] memory amounts = new uint256[](2);
        amounts[0] = 100 ether;
        amounts[1] = 100 ether;

        (uint256 totalFee, uint256[] memory newAmounts) =
            helper.exposedCalculateFeeAndNewAmountForOneTVS(1_000, amounts);

        assertEq(totalFee, 20 ether, "aggregate fee should be 20%");
        assertEq(newAmounts[0], 90 ether, "first flow should only lose its own 10%");
        assertEq(newAmounts[1], 80 ether, "second flow wrongly loses cumulative 20%");
    }
}

contract FeeHelperHarness is FeesManager {
    function exposedCalculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts)
        external
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
        uint256 length = amounts.length;
        newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
            unchecked {
                ++i;
            }
        }
    }
}

```

### Mitigation

Compute the per-flow fee once (`uint256 fee = calculateFeeAmount(...);`) and subtract that single value from the corresponding flow (`newAmounts[i] = amounts[i] - fee`). Do not reuse the cumulative `feeAmount` variable in the subtraction.
  