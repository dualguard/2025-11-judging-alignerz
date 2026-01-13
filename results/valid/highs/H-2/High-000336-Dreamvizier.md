# [000336] Users cannot split or merge TVS due to uninitialized array causing panic
  
  ### Summary

A missing array initialization in the fee calculation function will cause an array out-of-bounds panic when accessing the return array, preventing users from splitting or merging TVS as the function reverts immediately.

### Root Cause

In [FeesManager.sol:169](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169) the function `calculateFeeAndNewAmountForOneTVS` declares `uint256[] memory newAmounts` but never initializes it with a size:

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;  // PANIC: array out-of-bounds access (0x32)
    }
}
```

Accessing `newAmounts[i]` when the array has no size causes a panic with error code 0x32 (array out-of-bounds access).


### Internal Pre-conditions

1. User must have a TVS to split or merge
2. User must call `splitTVS()` or `mergeTVS()` which internally calls `calculateFeeAndNewAmountForOneTVS()`
3. The function must execute and attempt to access `newAmounts[0]` or any index


### External Pre-conditions

None

### Attack Path

1. User attempts to split or merge their TVS
2. `splitTVS()` or `mergeTVS()` calls `calculateFeeAndNewAmountForOneTVS()`
3. Function enters loop with `i = 0`
4. Function attempts to access `newAmounts[0]` to assign a value
5. Array `newAmounts` has no size (length = 0)
6. Accessing index 0 on an array with no size causes panic 0x32
7. Transaction reverts immediately
8. User cannot complete split or merge operation


### Impact

Users cannot split or merge TVS as the operations will always revert with a panic error. This is a denial of service that completely breaks the split and merge functionality. While funds are not directly locked (users can still claim tokens normally), the inability to split or merge TVS prevents users from managing their allocations as intended. Admin intervention via contract upgrade would be required to restore functionality.

### PoC

**PLEASE NOTE**: The `calculateFeeAndNewAmountForOneTVS` function is bugged initially due to the cumulative fee being subtracted instead of the relevant fee amount, and also due to the missing increment in the loop `i++` resulting in an infinite loop. This POC doesn't fix the function because it's unnecessary to showcase the current bug (the other ones are described in other submissions). Indeed, the PANIC happens on the first iteration, with index `0`


Run the coded POC:
```bash
forge test --match-path "test/UninitializedArrayBug.t.sol" -vv
```

The test demonstrates:
- Function with correct loop increment and fee calculation (assuming those bugs are fixed)
- Array is never initialized with size
- Accessing `newAmounts[0]` causes panic 0x32 (array out-of-bounds access)
- Transaction reverts immediately

Screenshot of the execution:

<img width="618" height="229" alt="Image" src="https://github.com/user-attachments/assets/167cfc74-d2a3-49c4-bd32-c0175ab2f906" />

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import {Test, console} from "forge-std/Test.sol";

/**
 * @title UninitializedArrayBug
 * @notice POC demonstrating array out-of-bounds panic due to uninitialized array
 * 
 * THE BUG:
 * The function never initializes the `newAmounts` array with a size, so accessing
 * any index (even `newAmounts[0]`) causes a panic: array out-of-bounds access (0x32).
 */
contract UninitializedArrayBug is Test {
    uint256 constant BASIS_POINT = 10_000;
    
    // Function with increment and correct fee calculation fixed, but array not initialized
    function calculateFeeAndNewAmountForOneTVS(
        uint256 feeRate, 
        uint256[] memory amounts, 
        uint256 length
    ) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        // BUG: newAmounts is never initialized with a size!
        // Should be: newAmounts = new uint256[](length);
        
        for (uint256 i; i < length;) {
            uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
            feeAmount += fee;
            newAmounts[i] = amounts[i] - fee;  // PANIC: array out-of-bounds access
            unchecked { ++i; }
        }
    }
    
    function calculateFeeAmount(uint256 feeRate, uint256 amount) public pure returns(uint256 feeAmount) {
        feeAmount = amount * feeRate / BASIS_POINT;
    }
    
    /**
     * @notice This test demonstrates the array out-of-bounds panic
     */
    function test_UninitializedArrayCausesPanic() public {
        uint256 feeRate = 50;
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 1000 * 1e18;
        amounts[1] = 2000 * 1e18;
        amounts[2] = 3000 * 1e18;
        
        console.log("Calling function with uninitialized array");
        console.log("This will panic with array out-of-bounds access");
        
        // This will panic with 0x32 (array out-of-bounds access)
        // Panic code 0x32 = array out-of-bounds access
        vm.expectRevert(abi.encodeWithSelector(0x4e487b71, 0x32));
        this.calculateFeeAndNewAmountForOneTVS(feeRate, amounts, amounts.length);
    }
    
    /**
     * @notice Show what happens when accessing uninitialized array
     */
    function test_UninitializedArrayAccess() public pure {
        uint256[] memory arr;
        // arr.length is 0
        // arr[0] would panic with 0x32
        
        // This demonstrates the issue - can't even access index 0
        // because the array has no size
    }
}


```

### Mitigation

Initialize the array with the correct size at the start of the function:

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    newAmounts = new uint256[](length);  // Fix: Initialize array with size
    for (uint256 i; i < length;) {
        uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
        feeAmount += fee;
        newAmounts[i] = amounts[i] - fee;
        unchecked { ++i; }
    }
}
```
  