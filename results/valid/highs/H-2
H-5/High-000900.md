# [000900] "Fee helper defects will block merges/splits and misprice vesting flows for all users"
  
  ### Summary

Uninitialized/write-into-zero-length logic and cumulative subtraction in `calculateFeeAndNewAmountForOneTVS` will cause a hard revert (or mispriced fees if fixed) for all vesting users as any caller of `mergeTVS`/`splitTVS` will hit an out-of-bounds write before state updates.

### Root Cause

In `src/contracts/vesting/feesManager/FeesManager.sol:83-90` the helper writes to `newAmounts[i]` without allocating `newAmounts` and never increments `i`, so the first iteration panics with an out-of-bounds access. Even if allocation/indexing were fixed, the code subtracts the cumulative fee instead of the per-flow fee, over-deducting later flows.

### Internal Pre-conditions

1. Owner sets non-zero `splitFeeRate`/`mergeFeeRate` (typical configuration).
2. A user holds a TVS NFT with at least one flow and invokes `mergeTVS` or `splitTVS` (no special sequencing required).

### External Pre-conditions

None.

### Attack Path

1. User calls `mergeTVS` or `splitTVS` on any TVS (bidding or reward).
2. The function calls `calculateFeeAndNewAmountForOneTVS` with a non-empty `amounts` array.
3. Inside the helper, the first write to `newAmounts[0]` reverts because `newAmounts` is length 0 (and `i` never increments), halting the transaction before any state change or fee transfer.
4. If the helper were patched to allocate but not to fix the logic, later flows would have the cumulative fee subtracted, overcharging and risking underflow/value loss.

### Impact

The users cannot execute any merge or split operations (feature freeze), and the treasury cannot collect merge/split fees. If only the revert were fixed without correcting the cumulative subtraction, users would be overcharged on later flows, losing vesting value to the treasury or triggering underflow reverts. Breaking core functionality

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {FeesManager} from "../src/contracts/vesting/feesManager/FeesManager.sol";

/// @dev Minimal harness to invoke the existing fee helper.
contract FeesManagerHarness is FeesManager {
    function callCalculate(uint256 feeRate, uint256[] memory amounts)
        external
        pure
        returns (uint256, uint256[] memory)
    {
        return calculateFeeAndNewAmountForOneTVS(feeRate, amounts, amounts.length);
    }
}

contract FeesManagerCalculateFeeBugTest is Test {
    FeesManagerHarness private harness;

    function setUp() public {
        harness = new FeesManagerHarness();
    }

    /// The helper writes into an unallocated array and never increments the loop index,
    /// so any non-empty input reverts with an out-of-bounds panic before it can return.
    function test_calculateFeeAndNewAmountForOneTVS_alwaysReverts() public {
        uint256[] memory amounts = new uint256[](2);
        amounts[0] = 1 ether;
        amounts[1] = 2 ether;

        vm.expectRevert(stdError.indexOOBError);
        harness.callCalculate({feeRate: 100, amounts: amounts});
    }
}
```

### Mitigation

- Allocate `newAmounts = new uint256[](length);` and increment `i` with unchecked `++i`.
- Use per-flow fee: `fee_i = calculateFeeAmount(feeRate, amounts[i]); newAmounts[i] = amounts[i] - fee_i; feeAmount += fee_i;`.
- Fix `_computeSplitArrays` to allocate its arrays before writing.
- Add invariant tests ensuring Σ(new) + fee == Σ(old) and non-reverting behavior for >0 flows.
  