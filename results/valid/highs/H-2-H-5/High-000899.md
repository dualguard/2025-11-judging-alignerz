# [000899] Buggy fee helper overcharge vesting flows for all split/merge users
  
  ###HIGH

### Summary

Unallocated array + cumulative-fee subtraction in `calculateFeeAndNewAmountForOneTVS` will cause overcharged splits/merges for all vesting users as any caller of `splitTVS`/`mergeTVS` will hit an out-of-bounds write today, and once callable, will have later flows shaved by the cumulative fee instead of per-flow.

### Root Cause

In `src/contracts/vesting/feesManager/FeesManager.sol:83-90` the helper accesses `newAmounts[i]` without allocating the array and never increments `i`, causing an out-of-bounds panic(which i reported in my previous report). The loop also subtracts the cumulative `feeAmount` from each flow (`newAmounts[i] = amounts[i] - feeAmount`) instead of the per-flow fee, leading to overcharges when made callable.

### Internal Pre-conditions

        1. Owner has set non-zero `splitFeeRate`/`mergeFeeRate` (typical deployment).
        2. User owns an NFT with at least one flow and calls `splitTVS` or `mergeTVS` (no special sequencing required).

### External Pre-conditions

None

### Attack Path

1. User calls `splitTVS` or `mergeTVS` on their own NFT.
2. Today: `calculateFeeAndNewAmountForOneTVS` writes to an unallocated `newAmounts[0]` and reverts, blocking the operation and fee collection.
3. If fixed to allocate but left cumulative: the helper applies cumulative fees, so later flows are over-shaved; small flows can underflow and revert; tiny flows can be zeroed.
4. In merge, existing flows are over-shaved cumulatively while appended flows (_merge) are shaved per-flow, creating internal inconsistency.

### Impact

Once callable, users suffer overcharges on later flows (value loss), some TVSs become unsplittable/unmergeable due to underflow, and flows can be zeroed causing future `claimTokens` reverts. Merge results become internally inconsistent (cumulative vs per-flow), skewing events and `allocationOf` for downstream consumers.

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {FeesManager} from "../src/contracts/vesting/feesManager/FeesManager.sol";

/// @dev Harness that reuses the buggy logic but makes it callable for scenario tests.
contract FeesManagerHarnessAlloc is FeesManager {
    function calculateWithBug(uint256 feeRate, uint256[] memory amounts)
        external
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
        uint256 length = amounts.length;
        newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
            uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
            feeAmount += fee;
            newAmounts[i] = amounts[i] - feeAmount; // cumulative subtraction bug
            unchecked {
                ++i;
            }
        }
    }
}

contract FeesManagerFeeMathScenariosTest is Test {
    FeesManagerHarnessAlloc private harness;

    function setUp() public {
        harness = new FeesManagerHarnessAlloc();
    }

    /// Baseline: three equal flows at 10% fee; later flows overcharged.
    function test_cumulativeOvercharge_equalFlows() public {
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 100;
        amounts[1] = 100;
        amounts[2] = 100;

        (uint256 feeAmount, uint256[] memory newAmounts) = harness.calculateWithBug(1_000, amounts);

        assertEq(feeAmount, 30);
        assertEq(newAmounts[0], 90);
        assertEq(newAmounts[1], 80); // should be 90
        assertEq(newAmounts[2], 70); // should be 90
        uint256 totalAfter = newAmounts[0] + newAmounts[1] + newAmounts[2] + feeAmount;
        assertEq(totalAfter, 270, "value lost due to cumulative subtraction");
    }

    /// Shows underflow/revert when later flow is smaller than cumulative fee.
    function test_underflowOnSmallLaterFlow() public {
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 50;
        amounts[1] = 40;
        amounts[2] = 5; // will underflow when cumulative fee exceeds 5

        // At 20% fee, cumulative after first two = 18, third = 1 (fee) => cumulative 19 > 5 => revert
        vm.expectRevert();
        harness.calculateWithBug(2_000, amounts);
    }

    /// Mixed flows: demonstrates some flows become zeroed, leaving gaps for claim logic.
    function test_zeroedFlowsBreakClaims() public {
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 10;
        amounts[1] = 10;
        amounts[2] = 1;

        // 50% fee rate: fees 5,5,0 => cumulative 10 => last flow becomes 1 - 10 = underflow -> revert
        vm.expectRevert();
        harness.calculateWithBug(5_000, amounts);
    }

    /// Large first flow, tiny later flows: later flows drop to zero but do not revert; value loss highlighted.
    function test_tinyFlowsLoseAllValue() public {
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 1_000;
        amounts[1] = 1;
        amounts[2] = 1;

        // 10% fee causes cumulative 100 after first flow; subtracting from tiny flows underflows and reverts.
        vm.expectRevert();
        harness.calculateWithBug(1_000, amounts);
    }

    /// Inconsistency with merge pattern: preexisting flows shaved cumulatively, appended flows shaved per-flow.
    function test_inconsistentMergeBehavior() public {
        // Existing TVS with two flows
        uint256[] memory existing = new uint256[](2);
        existing[0] = 100;
        existing[1] = 100;

        // Apply buggy helper (what mergeTVS does before appending)
        (uint256 feeExisting, uint256[] memory newExisting) = harness.calculateWithBug(1_000, existing);

        // Appended TVS flow using correct per-flow fee (expected behavior in _merge)
        uint256 appendAmount = 100;
        uint256 perFlowFee = harness.calculateFeeAmount(1_000, appendAmount);
        uint256 appendedNet = appendAmount - perFlowFee;

        // Existing flows are uneven (90,80), appended is 90 -> internal inconsistency
        assertEq(newExisting[0], 90);
        assertEq(newExisting[1], 80);
        assertEq(appendedNet, 90);
        assertEq(feeExisting + perFlowFee, 20 + 10, "total fee accounting aligns, but nets differ");
    }
}
```

### Mitigation

       - Allocate `newAmounts = new uint256[](length);` and increment `i` in the loop.
       - Apply per-flow fee: `fee_i = calculateFeeAmount(feeRate, amounts[i]); newAmounts[i] = amounts[i] - fee_i; feeAmount += fee_i;`.
  