# [000226] [Medium] Fee helper lacks array allocation so merge/split flows always revert
  
  ### Summary

The helper `calculateFeeAndNewAmountForOneTVS` writes into an unallocated named return array and never increments its loop counter, which causes every call to `mergeTVS` or `splitTVS` to revert before any amounts are adjusted.

### Root Cause

In [protocol/src/contracts/vesting/feesManager/FeesManager.sol:169-174](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174) the function returns `(uint256 feeAmount, uint256[] memory newAmounts)` but never initializes `newAmounts` (length stays zero) and omits `++i` in the loop. The first assignment to `newAmounts[i]` panics with an out-of-bounds error, preventing `AlignerzVesting` from restructuring TVSs.

### Internal Pre-conditions

1. At least one TVS allocation exists with `amounts.length > 0` (true for any minted NFT).
2. Split or merge fees are configured (any non-negative value triggers the helper).
3. A user calls `splitTVS` or `mergeTVS`, which invokes `calculateFeeAndNewAmountForOneTVS`.

### External Pre-conditions

None

### Attack Path

1. User owns an NFT whose `allocation.amounts[0] = 100 ether`.
2. User calls `splitTVS` or `mergeTVS`.
3. `AlignerzVesting` calls `calculateFeeAndNewAmountForOneTVS` with `length = 1`.
4. [Line 171](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L171) accumulates the fee into `feeAmount`, but `newAmounts` is still zero-length.
5. [Line 172](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L172) executes `newAmounts[0] = …`, triggering a panic and reverting the entire transaction before any state updates.

### Impact

TVS holders cannot split or merge their allocations, and the protocol cannot collect the intended fees, rendering all lifecycle management features unusable.

### PoC

1. Create test file `FeesManager.t.sol` and place in `protocol/test/` folder.
2. Copy code below into `FeesManager.t.sol`.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.29;

import "forge-std/Test.sol";
import {FeesManager} from "../src/contracts/vesting/feesManager/FeesManager.sol";

contract FeesManagerTest is Test {
    FeesManagerHarness internal harness;

    function setUp() public {
        harness = new FeesManagerHarness();
    }

    function testCalculateFeeAndNewAmountForOneTVSAlwaysRevertsDueToMissingArrayAllocation() public {
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 100 ether;

        vm.expectRevert();
        harness.exposedCalculate(amounts);
    }
}

contract FeesManagerHarness is FeesManager {
    function exposedCalculate(uint256[] memory amounts)
        external
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
        return calculateFeeAndNewAmountForOneTVS(100, amounts, amounts.length);
    }
}
```
3. Run `forge test --match-test testCalculateFeeAndNewAmountForOneTVSAlwaysRevertsDueToMissingArrayAllocation`.

### Mitigation

Allocate the return array (`uint256[] memory newAmounts = new uint256[](length);`) and increment `i` on each iteration so every flow produces a properly fee-adjusted amount without panicking.
  