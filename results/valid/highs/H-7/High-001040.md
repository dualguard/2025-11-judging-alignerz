# [001040] Protocol will be completely unusable due to infinite loop in FeeManager::calculateFeeAndNewAmountForOneTVS()
  
  ## Summary

Missing loop increment in fee calculation will cause complete denial of service for protocol users as user will trigger infinite loop during merge or split operations
https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L175

## Root Cause

In `src/contracts/vesting/feesManager/FeesManager.sol:170-173` the for loop is missing the increment statement, causing the loop variable to never increase, resulting in an infinite loop that consumes all gas and reverts transactions.
```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```
## Internal Pre-conditions

1. Protocol owner needs to set non-zero merge or split fee rates
2. TVS allocation needs to have at least 1 flow

## External Pre-conditions

None - this is a deterministic infinite loop that always triggers when fee calculation is called.

## Attack Path

This is a vulnerability path that completely breaks core functionality:

1. **User calls `mergeTVS()` or `splitTVS()`** to manage their TVS allocations
2. **Contract executes and reaches the fee calculation** at `src/contracts/vesting/AlignerzVesting.sol:1013` or `1069`
3. **Function calls `calculateFeeAndNewAmountForOneTVS()`** to compute fees
4. **Loop executes with i = 0**:
   - Iteration 1: Calculates fee for `amounts[0]`, stores in `newAmounts[0]`
   - Loop checks: `i < length` → `0 < length` → TRUE
   - **Loop repeats**: i is still 0 (never incremented!)
   - Iteration 2: Same calculation for `amounts[0]` again
   - Loop checks: `i < length` → `0 < length` → TRUE
   - **Infinite loop continues** until gas runs out
5. **Transaction reverts with out-of-gas error**
6. **User cannot merge or split TVS** - functionality is completely broken

## Impact

The protocol users cannot merge or split TVS allocations. All transactions attempting to merge or split will fail with out-of-gas errors, regardless of gas limit set.

Affected functions:
- **`mergeTVS()` at src/contracts/vesting/AlignerzVesting.sol:1002** - completely broken, infinite loop on line 1013
- **`splitTVS()` at src/contracts/vesting/AlignerzVesting.sol:1054** - completely broken, infinite loop on line 1069

This is a catastrophic deployment-blocking bug that makes core protocol features completely non-functional.

## PoC

**Test code:**

> NOTE: You need to fix uninitialized `newAmounts` array and `feeAmounts` calculation bugs I reported in other findings for this to work.

Add this test to `test/AlignerzVestingProtocolTest.t.sol`:

```solidity
/// @notice Test demonstrating infinite loop in calculateFeeAndNewAmountForOneTVS
function test_InfiniteLoopInFeeCalculation() public {
    vm.startPrank(projectCreator);
    vesting.setMergeFeeRate(200); // 2% fee

    uint256[] memory amounts = new uint256[](1);
    amounts[0] = 1000 ether;

    console.log("Calling calculateFeeAndNewAmountForOneTVS with 1 flow...");
    console.log("Expected: Should complete immediately and return");
    console.log("Actual: Will hang in infinite loop and consume all gas");

    vm.expectRevert(); // Expecting out-of-gas revert

    // This call will never complete - infinite loop
    (uint256 feeAmount, uint256[] memory newAmounts) = vesting.calculateFeeAndNewAmountForOneTVS(
        vesting.mergeFeeRate(),
        amounts,
        amounts.length
    );

    console.log("This line will never be reached!");
}
```

## Mitigation

Add the missing loop increment in `src/contracts/vesting/feesManager/FeesManager.sol:170-173`:

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
    public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    newAmounts = new uint256[](length);
    for (uint256 i; i < length;) {
        uint256 individualFee = calculateFeeAmount(feeRate, amounts[i]);
        feeAmount += individualFee;
        newAmounts[i] = amounts[i] - individualFee;
        unchecked {
            ++i;  // Add loop increment
        }
    }
}
```

This ensures:
1. Loop variable `i` is incremented after each iteration
2. Loop eventually reaches `i >= length` and exits
3. Function completes successfully
4. Merge and split operations work as intended

The `unchecked` block is used to save gas since `i` cannot realistically overflow in this context (array lengths are bounded by memory constraints).
```
  