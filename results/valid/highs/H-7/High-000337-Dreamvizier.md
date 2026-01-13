# [000337] Users cannot split or merge TVS due to infinite loop in fee calculation
  
  ### Summary

A missing loop increment in the fee calculation function will cause an infinite loop that exhausts gas, preventing users from splitting or merging TVS as the function never terminates.

### Root Cause

In [FeesManager.sol:170](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L170) the loop in `calculateFeeAndNewAmountForOneTVS` never increments the loop variable `i`, causing an infinite loop:

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
        // Missing: unchecked { ++i; }
    }
}
```

The loop condition `i < length` will always be true since `i` never increments, causing the loop to run indefinitely until gas is exhausted.

### Internal Pre-conditions

1. User must have a TVS to split or merge
2. User must call `splitTVS()` or `mergeTVS()` which internally calls `calculateFeeAndNewAmountForOneTVS()`
3. The function must execute (any call will trigger the infinite loop)

### External Pre-conditions

None

### Attack Path

1. User attempts to split or merge their TVS
2. `splitTVS()` or `mergeTVS()` calls `calculateFeeAndNewAmountForOneTVS()`
3. Function enters loop with `i = 0` and `length > 0`
4. Loop executes first iteration but `i` never increments
5. Loop condition `i < length` remains true forever
6. Loop continues indefinitely until gas limit is reached
7. Transaction reverts with out of gas error
8. User cannot complete split or merge operation

### Impact

Users cannot split or merge TVS as the operations will always revert due to gas exhaustion. This is a denial of service that completely breaks the split and merge functionality. While funds are not directly locked (users can still claim tokens normally), the inability to split or merge TVS prevents users from managing their allocations as intended. Admin intervention via contract upgrade would be required to restore functionality.

### PoC

**PLEASE NOTE**: The `calculateFeeAndNewAmountForOneTVS` function is bugged initially due to the cumulative fee being subtracted instead of the relevant fee amount, and also due to the return array `uint256[] memory newAmounts` not being initialized. This POC fixes the function to showcase only the current bug and the other ones are described in other submissions.

Run the coded POC:
```bash
forge test --match-path "test/InfiniteLoopDoS.t.sol" -vv
```

The test demonstrates:
- The loop variable never increments
- Function would run out of gas if called
- The infinite loop condition

Screenshot of the execution:

<img width="568" height="897" alt="Image" src="https://github.com/user-attachments/assets/2a5655b1-c150-49cb-86f2-1cc9bb71529c" />

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import {Test, console} from "forge-std/Test.sol";

/**
 * @title InfiniteLoopDoS
 * @notice POC demonstrating infinite loop in calculateFeeAndNewAmountForOneTVS
 * 
 * THE BUG:
 * The loop in calculateFeeAndNewAmountForOneTVS never increments `i`, causing
 * an infinite loop that will always run out of gas.
 */
contract InfiniteLoopDoS is Test {
    uint256 constant BASIS_POINT = 10_000;
    
    // Fixed copy of the buggy function from FeesManager.sol
    function fixedCopyCalculateFeeAndNewAmountForOneTVS(
        uint256 feeRate, 
        uint256[] memory amounts, 
        uint256 length
    ) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint256[](length);//@audit FIX FROM OTHER BUG: the array wasn't initialize and resulted in a panic out-of-bound access
        for (uint256 i; i < length;) {
            uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
            feeAmount += fee;
            newAmounts[i] = amounts[i] - fee;  //@audit FIX FROM OTHER BUG: The cumulative amount (another finding) would be the reason for DOS in the infinite loop so we fix here to subtract only this amount's fee
            console.log("We're inside the loop"); //@audit  BUG: No increment! Loop never terminates
        }
    }
    
    function calculateFeeAmount(uint256 feeRate, uint256 amount) public pure returns(uint256 feeAmount) {
        feeAmount = amount * feeRate / BASIS_POINT;
    }
    
    /**
     * @notice This test demonstrates the infinite loop by showing it would run out of gas
     */
    function test_InfiniteLoopRunsOutOfGas() public {
        uint256 feeRate = 50;
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 1000 * 1e18;
        amounts[1] = 2000 * 1e18;
        amounts[2] = 3000 * 1e18;
        
        console.log("Calling buggy function - this will run out of gas");
        console.log("The loop never increments i, so it runs forever");
        
        // This will revert with out of gas due to infinite loop
        
        vm.expectRevert();
        this.fixedCopyCalculateFeeAndNewAmountForOneTVS(feeRate, amounts, amounts.length);
    }
}


```

### Mitigation

Add the missing loop increment:

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    newAmounts = new uint256[](length);
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
        unchecked { ++i; }  // Fix: Increment loop variable
    }
}
```
  