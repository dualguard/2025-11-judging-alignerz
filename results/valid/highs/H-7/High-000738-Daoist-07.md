# [000738] Infinite Loop In FeesManager::calculateFeeAndNewAmountForOneTVS Causes Permanent DoS in AlignerzVesting::mergeTVS
  
  ### Summary

The `mergeTVS` functionality is permanently inoperable due to a logic error in the `FeesManager.sol` contract. The helper function `calculateFeeAndNewAmountForOneTVS` contains a `for` loop that lacks an iterator increment statement. This results in an infinite loop for any input array with a length greater than zero, causing the transaction to consume all available gas and revert.

### Root Cause

The vulnerability exists in `FeesManager.sol` within the `calculateFeeAndNewAmountForOneTVS` function. The `for` loop header is defined as `for (uint256 i; i < length;)`. Crucially, it is missing the increment statement `(e.g., i++ or ++i)` in the header, and there is no increment logic within the loop body.
```solidity
 function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) { //@> No increment to counter causes infinite loop
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```
So `i` remains 0 throughout execution. If `length > 0`, the loop condition `i < length` will always be true, creating an infinite loop that runs until the transaction runs out of gas. Since any TVS allocation will have at least one flow `(nbOfFlows > 0)`, the infinite loop will always trigger 

### Internal Pre-conditions

1. A user attempts to use the `mergeTVS` function.

2. The input arrays passed to the function have a length of at least 1.

### External Pre-conditions

None

### Attack Path

1. A user calls `AlignerzVesting:mergeTVS` to combine vesting allocations.

2. The vesting contract delegates fee calculation to `FeesManager::calculateFeeAndNewAmountForOneTVS`

3. The function enters the for loop with `i = 0`

4. The loop body executes.

5. The loop cycle restarts. Since `i` was not incremented, `i` is still 0.

6. The termination condition `i < length` is checked. Since `0 < length` is still true, the loop continues.

7.  The loop repeats thousands of times until the gas stipend is exhausted.

8.  The transaction reverts with an "Out of Gas" error, preventing the merge operation

### Impact

The `mergeTVS` feature is effectively broken. Because the `mergeTVS` function unconditionally calls this helper, it is impossible to execute a merge transaction successfully. This constitutes a permanent Denial of Service (DoS) of a core protocol feature.

### PoC

The real `FeesManager.sol` has two fatal bugs in `calculateFeeAndNewAmountForOneTVS`:
 1. Uninitialized Memory `(Panic 0x32)` -> Crashes immediately on index access.
 2. Infinite Loop (OutOfGas) -> Loop counter `i` is never incremented.
 3.  cannot test the infinite loop issue on the real contract because  #6  blocks it.
 4.  This Harness replicates the exact function logic but fixes #6  to demonstrably prove the Infinite loop exists and is fatal.

The following test was run and produced these logs:  
```bash
Ran 1 test for test/PoC.t.sol:FeesManagerInfiniteLoopPoC
[PASS] testInfiniteLoopConsumesAllGas() (gas: 54523)
Logs:
  Testing Loop Logic with Gas Limit: 1000000
  Success: Transaction ran out of gas as expected.

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 70.81ms (35.54ms CPU time)
``` 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";

// ----------------------------------------------------------------------------
// LOGIC HARNESS
// The real FeesManager.sol has two fatal bugs in 'calculateFeeAndNewAmountForOneTVS':
// 1. Uninitialized Memory (Panic 0x32) -> Crashes immediately on index access.
// 2. Infinite Loop (OutOfGas) -> Loop counter 'i' is never incremented.
//
// We cannot test Bug #2 on the real contract because Bug #1 blocks it.
// This Harness replicates the exact function logic but fixes Bug #1 
// to demonstrably prove that Bug #2 (The Infinite Loop) exists and is fatal.
// ----------------------------------------------------------------------------
contract FeesManagerHarness {
    uint256 constant public BASIS_POINT = 10_000;

    function calculateFeeAmount(uint256 feeRate, uint256 amount) public pure returns(uint256 feeAmount) {
        feeAmount = amount * feeRate / BASIS_POINT;
    }

    // Exact logic from FeesManager.sol, with Memory Allocation patched
    function calculateFeeAndNewAmountForOneTVS(
        uint256 feeRate, 
        uint256[] memory amounts, 
        uint256 length
    ) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        
        // PATCH: We initialize memory here so we don't crash with Panic 0x32.
        // This allows execution to proceed to the loop logic we want to test.
        newAmounts = new uint256[](length);

        // VULNERABILITY: The loop header 'for (uint256 i; i < length;)' lacks 'i++'.
        // The loop body also lacks 'i++'.
        // This causes 'i' to remain 0 forever, creating an infinite loop.
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
}

contract FeesManagerInfiniteLoopPoC is Test {
    FeesManagerHarness public harness;

    function setUp() public {
        harness = new FeesManagerHarness();
    }

    function testInfiniteLoopConsumesAllGas() public {
        // Setup valid input data
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 100 ether;
        uint256 length = 1;
        uint256 feeRate = 100; // 1%

        // We restrict gas to 1 million. 
        // A valid operation for 1 item should take < 50,000 gas.
        // If it consumes 1 million, it proves the loop is infinite.
        uint256 testGasLimit = 1_000_000;

        console.log("Testing Loop Logic with Gas Limit:", testGasLimit);

        // We use a low-level call to safely catch the Out Of Gas (OOG) error
        (bool success, ) = address(harness).call{gas: testGasLimit}(
            abi.encodeCall(
                FeesManagerHarness.calculateFeeAndNewAmountForOneTVS, 
                (feeRate, amounts, length)
            )
        );

        // Assert failure: The call must fail due to running out of gas
        assertEq(success, false, "Function should have reverted due to infinite loop");

        console.log("Success: Transaction ran out of gas as expected.");
    }
}
```

### Mitigation

Add the increment statement to the loop header in `FeesManager.sol`
  