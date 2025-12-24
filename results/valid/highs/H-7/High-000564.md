# [000564] Missing Loop Increment in `calculateFeeAndNewAmountForOneTVS` Causes Denial of Service for `mergeTVS` and `splitTVS` Functions
  
  
## Summary

A missing loop counter increment in `calculateFeeAndNewAmountForOneTVS` causes an infinite loop, leading to out-of-gas failures and denial of service for users and the protocol. Any call to `mergeTVS` or `splitTVS` that uses this function will revert due to gas exhaustion.

## Root Cause

In [`FeesManager.sol:170`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174), the `for` loop uses `for (uint256 i; i < length;)` without incrementing `i` in the body. The loop condition never becomes false, so it runs until gas is exhausted, causing all dependent functions to fail.
```js
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

## Internal Pre-conditions

1. A user must call `mergeTVS()` or `splitTVS()` with at least one flow in the TVS allocation
2. The `length` parameter passed to `calculateFeeAndNewAmountForOneTVS` must be greater than 0
3. The transaction must have sufficient gas to attempt execution (will fail regardless)

## External Pre-conditions

None

## Attack Path

1. A user calls `mergeTVS()` or `splitTVS()` with a TVS that has one or more flows
2. The function calls `calculateFeeAndNewAmountForOneTVS()` to compute fees
3. The loop starts with `i = 0` and checks `i < length`
4. The loop body executes but never increments `i`
5. The condition `i < length` remains true, so the loop continues indefinitely
6. The transaction runs out of gas and reverts
7. The user cannot complete the merge or split operation

## Impact

Users cannot execute `mergeTVS()` or `splitTVS()`, causing a denial of service. These functions are permanently disabled, blocking TVS management. The protocol cannot collect fees from these operations, and users cannot manage their vesting positions.

## Proof of Concept

```solidity
function testInfiniteLoopDoS() public {
    uint256 feeRate = 100; // 1% fee rate
    uint256[] memory amounts = new uint256[](3);
    amounts[0] = 1000;
    amounts[1] = 2000;
    amounts[2] = 3000;
    uint256 length = 3;
    
    // This call will run out of gas due to infinite loop
    // The loop counter i never increments, so i < length is always true
    (uint256 feeAmount, uint256[] memory newAmounts) = 
        calculateFeeAndNewAmountForOneTVS(feeRate, amounts, length);
    
    // This line is never reached
    assert(feeAmount > 0);
}

// In a real scenario:
// 1. User calls mergeTVS() with a TVS containing multiple flows
// 2. mergeTVS() calls calculateFeeAndNewAmountForOneTVS()
// 3. Function enters infinite loop at i=0
// 4. Transaction runs out of gas and reverts
// 5. User cannot merge their TVS
```

## Mitigation

Add the loop counter increment in the loop body. Use the unchecked block for gas optimization:

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    newAmounts = new uint256[](length);
    for (uint256 i; i < length;) {
        uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
        feeAmount += fee;
        newAmounts[i] = amounts[i] - fee;
        unchecked { ++i; }
    }
}
```

Alternatively, use a traditional for loop:

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    newAmounts = new uint256[](length);
    for (uint256 i = 0; i < length; i++) {
        uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
        feeAmount += fee;
        newAmounts[i] = amounts[i] - fee;
    }
}
```

Note: The fix also corrects the logic bug where `feeAmount` was accumulated but subtracted from each individual amount. The corrected version calculates the fee per amount and subtracts only that fee from each corresponding amount.
  