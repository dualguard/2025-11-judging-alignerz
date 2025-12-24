# [000552] Uninitialized Memory Arrays in `_computeSplitArrays` Cause Out-of-Bounds Access Panics and Prevent TVS Splitting
  
  
## Summary

Uninitialized dynamic memory arrays in `_computeSplitArrays` cause out-of-bounds access panics when writing to array indices. This breaks TVS splitting, preventing users from splitting their vesting allocations.
```js
    function _computeSplitArrays(...)
        internal
        view
        returns (
            Allocation memory alloc
        )
    {
        // ...
        for (uint256 j; j < nbOfFlows;) {
            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
            alloc.vestingPeriods[j] = baseVestings[j];
            alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
            alloc.claimedSeconds[j] = baseClaimed[j];
            alloc.claimedFlows[j] = baseClaimedFlows[j];
            // ...
        }
    }
```

## Root Cause

In [`AlignerzVesting.sol:1114-1142`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113-L1123), `_computeSplitArrays` creates an `Allocation memory alloc` struct but does not initialize its dynamic arrays (`amounts`, `vestingPeriods`, `vestingStartTimes`, `claimedSeconds`, `claimedFlows`) before writing to them at lines 1133-1137. In Solidity, memory arrays must be initialized with a size before use; uninitialized arrays have zero length, causing an out-of-bounds panic on the first write.

## Internal Pre-conditions

1. A user must call `splitTVS()` with a valid NFT that has at least one vesting flow

## External Pre-conditions

None required. The bug triggers based on function invocation with valid parameters.

## Attack Path

1. A user calls `splitTVS()` with a valid `projectId`, `percentages` array, and `splitNftId` that has vesting allocations
2. The function calls `_computeSplitArrays()` at line 1089 to compute allocation arrays for each split TVS
3. `_computeSplitArrays()` creates an `Allocation memory alloc` struct but does not initialize its dynamic arrays
4. The function attempts to write to `alloc.amounts[j]`, `alloc.vestingPeriods[j]`, `alloc.vestingStartTimes[j]`, `alloc.claimedSeconds[j]`, and `alloc.claimedFlows[j]` at lines 1133-1137
5. The transaction reverts with a panic error (0x32: array out-of-bounds access) on the first loop iteration when accessing any uninitialized array index

## Impact

Users cannot split TVSs, blocking a core feature. The `splitTVS()` function reverts immediately, preventing users from dividing their vesting allocations into multiple NFTs.

## Proof of Concept

We use a minimal version that just focuses on the bug. Add the following code in any test files to execute the test
```js
    // forge test --match-test test_TestArrayNeedInitialization -vv
    function test_TestArrayNeedInitialization() public {
        // Test that _computeSplitArrays reverts due to lack of newAmounts array initialization
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 100;
        amounts[1] = 200;
        amounts[2] = 300;
        
        // This should revert because newAmounts array is not initialized in the function
        vm.expectRevert();
        this._computeSplitArrays(amounts, 3);
    }


    function _computeSplitArrays(uint256[] memory amounts, uint256 length) public pure returns (bool success, uint256[] memory newAmounts) {
        for (uint256 i; i < length; i++) {
            newAmounts[i] = amounts[i];
        }
        success = true;
    }
```

If you comment out the `vm.expectRevert()`, it fails with a panic revert
```bash
[FAIL: panic: array out-of-bounds access (0x32)] test_TestArrayNeedInitialization() (gas: 4062)
```



## Mitigation
Initialize all dynamic arrays in the `alloc` struct before the loop in `_computeSplitArrays`.

  