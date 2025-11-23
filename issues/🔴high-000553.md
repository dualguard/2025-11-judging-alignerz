# [000553] Uninitialized Memory Array in `calculateFeeAndNewAmountForOneTVS` Causes Out-of-Bounds Access Panic and Breaks Fee Calculation
  
  

## Summary

An uninitialized memory array in `calculateFeeAndNewAmountForOneTVS` causes an out-of-bounds access panic when writing to array indices. This breaks fee calculation for TVS splitting, preventing users from splitting TVSs and blocking fee calculations used in split operations.
```js
    function calculateFeeAndNewAmountForOneTVS(...) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            // ...
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

## Root Cause

In [`FeesManager.sol:169-174`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174), `calculateFeeAndNewAmountForOneTVS` declares `newAmounts` as a return parameter but never initializes it with `new uint256[](length)` before writing to `newAmounts[i]` at line 172. In Solidity, uninitialized memory arrays have zero length, causing an out-of-bounds panic on the first write.

## Internal Pre-conditions

1. A user or contract must call `calculateFeeAndNewAmountForOneTVS()` with a `length` parameter greater than zero, OR
2. A user must call `splitTVS()` or `mergeTVS()` which internally calls `calculateFeeAndNewAmountForOneTVS()` of `AlignerzVesting.sol`

## External Pre-conditions

None required. The bug triggers based on function invocation with valid parameters.

## Attack Path

1. A user calls `splitTVS()` or `mergeTVS()` with a valid `projectId`, `percentages` array, and `splitNftId` that has vesting allocations
2. The function calls `calculateFeeAndNewAmountForOneTVS()` to calculate fees and new amounts after applying the split fee
3. `calculateFeeAndNewAmountForOneTVS()` attempts to write to uninitialized `newAmounts[i]` at line 172
4. The transaction reverts with a panic error (0x32: array out-of-bounds access) on the first loop iteration when `i = 0`
5. The `splitTVS()` function fails, preventing the user from splitting their TVS

## Impact

Users cannot split TVSs, blocking a core feature. The `splitTVS()` function reverts immediately when calculating fees, preventing users from dividing their vesting allocations. This also affects any functionality that depends on `calculateFeeAndNewAmountForOneTVS()`.

## Proof of Concept

```solidity
// forge test --match-test test_TestArrayNeedInitialization -vv

function test_TestArrayNeedInitialization() public {
    // Test that _calculateFeeAndNewAmountForOneTVS reverts due to lack of newAmounts array initialization
    uint256[] memory amounts = new uint256[](3);
    amounts[0] = 100;
    amounts[1] = 200;
    amounts[2] = 300;
    
    // This should revert because newAmounts array is not initialized in the function
    vm.expectRevert();
    this._calculateFeeAndNewAmountForOneTVS(amounts, 3);
}

function _calculateFeeAndNewAmountForOneTVS(uint256[] memory amounts, uint256 length) public pure returns (bool success, uint256[] memory newAmounts) {
    for (uint256 i; i < length; i++) {
        newAmounts[i] = amounts[i];
    }
    success = true;
}
```

If you comment out the `vm.expectRevert()`, it fails with a panic revert:

```bash
[FAIL: panic: array out-of-bounds access (0x32)] test_TestArrayNeedInitialization() (gas: 4062)
```

## Mitigation

Initialize the `newAmounts` array at the beginning of the function in `FeesManager.sol:169`:

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    newAmounts = new uint256[](length);
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
        unchecked {
            ++i;
        }
    }
}
```
  