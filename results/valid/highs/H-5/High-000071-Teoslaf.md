# [000071] Incorrect fee logic overcharges users
  
  ## Summary

The [`calculateFeeAndNewAmountForOneTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174) function uses cumulative fees instead of individual fees when calculating new amounts for each flow. This causes users to lose progressively more tokens with each additional flow in their TVS. For a TVS with 5 flows totaling 75,000 tokens at 1% fee, users lose 1,000 tokens beyond the intended 750 token fee - a 133% overcharge.

Note: This bug is currently masked by an array initialization bug and infinite loop bug, but will cause fund loss once that bug is fixed.

See separate findings: 

1. `Uninitialized Array in calculateFeeAndNewAmountForOneTVS() Causes Complete DoS` 
2. `Infinite Loop DoS `

## Vulnerability Details

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
    public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);  // Accumulates total fee
        newAmounts[i] = amounts[i] - feeAmount;  // <- bug: Subtracts cumulative total instead of individual fee
    }
}
```

### Root Cause

The function correctly accumulates the total fee across all flows in `feeAmount`, but incorrectly uses this cumulative total when calculating each individual flow's new amount.

Each flow should only have its individual fee subtracted, not the cumulative total of all previous fees, and the more iteration it has the more tokens it loses.

Example: 5 flows [5000, 10000, 15000, 20000, 25000] tokens at 1% fee

| Iteration | Amount | Individual Fee | Cumulative Fee | Current Code (wrong)    | Should Be (correct)     | Extra Loss |
| --------- | ------ | -------------- | -------------- | ----------------------- | ----------------------- | ---------- |
| 0         | 5000   | 50             | 50             | `5000 - 50 = 4950` ✓    | `5000 - 50 = 4950` ✓    | 0          |
| 1         | 10000  | 100            | 150            | `10000 - 150 = 9850` ✗  | `10000 - 100 = 9900` ✓  | 50         |
| 2         | 15000  | 150            | 300            | `15000 - 300 = 14700` ✗ | `15000 - 150 = 14850` ✓ | 150        |
| 3         | 20000  | 200            | 500            | `20000 - 500 = 19500` ✗ | `20000 - 200 = 19800` ✓ | 300        |
| 4         | 25000  | 250            | 750            | `25000 - 750 = 24250` ✗ | `25000 - 250 = 24750` ✓ | 500        |
| Total     | 75000  | 750            | 750            | 73250                   | 74250                   | 1000       |

Result:

- Expected: User gets 74,250 tokens, treasury gets 750 tokens (total: 75,000 ✓)
- Actual: User gets 73,250 tokens, treasury gets 750 tokens (total: 74,000 ✗)
- Missing: 1,000 tokens disappeared (neither with user nor treasury)

## Proof of Concept

Add this test to demonstrate the bug:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";

contract IncorrectFeeTest is Test {
    uint256 constant BASIS_POINT = 10_000;

    // Helper function to calculate individual fee
    function calculateFeeAmount(uint256 feeRate, uint256 amount) public pure returns(uint256 feeAmount) {
        feeAmount = amount * feeRate / BASIS_POINT;
    }

    // buggy: Current implementation from the contract
    // Changes made to make it compile:
    // 1. Added: newAmounts = new uint256[](length) - fixes array initialization bug
	// 2. incriment i at the end of the loop
    // Bug that remains: uses cumulative feeAmount instead of individual fee per flow
    function current_calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) 
        public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint256[](length); // change: Added to fix Bug #1 (array not initialized)
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]); // Accumulates total fee
            newAmounts[i] = amounts[i] - feeAmount;  // BUG: Subtracts cumulative total, not individual fee
            unchecked { ++i; } // change: incriment i 
        }
    }

    // correct: Fixed implementation
    // Changes made to fix the cumulative fee bug:
    // 1. Added: newAmounts = new uint256[](length) - fixes array initialization bug
	// 2. incriment i at the end of the loop
    // 3. Subtract individual fee (not cumulative) from each amount
    function correct_calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) 
        public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
            uint256 individualFee = calculateFeeAmount(feeRate, amounts[i]); // FIX: Store individual fee
            feeAmount += individualFee; // Still accumulate total for return value
            newAmounts[i] = amounts[i] - individualFee;  // fix: Subtract individual, not cumulative
            unchecked { ++i; } // change: incriment i 
        }
    }

    // Poc with realistic vesting scenario: 5 flows, 75k tokens
    function test_CumulativeFeeBug_POC() public pure {
        uint256[] memory amounts = new uint256[](5);
        amounts[0] = 5_000 ether;
        amounts[1] = 10_000 ether;
        amounts[2] = 15_000 ether;
        amounts[3] = 20_000 ether;
        amounts[4] = 25_000 ether;
        uint256 feeRate = 100; // 1%

        // Call buggy implementation
        (uint256 buggyFee, uint256[] memory buggyAmounts) = 
            current_calculateFeeAndNewAmountForOneTVS(feeRate, amounts, 5);
        uint256 buggyTotal = 0;
        for (uint256 i = 0; i < 5; i++) buggyTotal += buggyAmounts[i];

        // Call correct implementation
        (uint256 correctFee, uint256[] memory correctAmounts) = 
            correct_calculateFeeAndNewAmountForOneTVS(feeRate, amounts, 5);
        uint256 correctTotal = 0;
        for (uint256 i = 0; i < 5; i++) correctTotal += correctAmounts[i];

        // Display results with clear explanation
        console.log("Input: 5 flows [5k, 10k, 15k, 20k, 25k] = 75k tokens\n");
        
        console.log("buggy (subtracts cumulative fee from each flow):");
        console.log("  Flow 0: 5000 - 50 = ", buggyAmounts[0]/1e18);
        console.log("  Flow 1: 10000 - 150 = ", buggyAmounts[1]/1e18, "(should subtract 100)");
        console.log("  Flow 2: 15000 - 300 = ", buggyAmounts[2]/1e18, "(should subtract 150)");
        console.log("  Flow 3: 20000 - 500 = ", buggyAmounts[3]/1e18, "(should subtract 200)");
        console.log("  Flow 4: 25000 - 750 = ", buggyAmounts[4]/1e18, "(should subtract 250)");
        console.log("  User gets:", buggyTotal/1e18, "tokens\n");

        console.log("correct (subtracts individual fee from each flow):");
        console.log("  Flow 0: 5000 - 50 = ", correctAmounts[0]/1e18);
        console.log("  Flow 1: 10000 - 100 = ", correctAmounts[1]/1e18);
        console.log("  Flow 2: 15000 - 150 = ", correctAmounts[2]/1e18);
        console.log("  Flow 3: 20000 - 200 = ", correctAmounts[3]/1e18);
        console.log("  Flow 4: 25000 - 250 = ", correctAmounts[4]/1e18);
        console.log("  User gets:", correctTotal/1e18, "tokens\n");

        uint256 loss = correctTotal - buggyTotal;
        console.log("result: User loses", loss/1e18, "tokens that disappear!\n");

        // Assertions
        uint256 totalInput = 75_000 ether;
        assertEq(buggyFee, correctFee, "Both collect same fee");
        assertEq(correctFee, 750 ether, "1% fee = 750 tokens");
        assertEq(correctTotal, 74_250 ether, "Correct: 74,250");
        assertEq(buggyTotal, 73_250 ether, "Buggy: 73,250");
        assertEq(loss, 1_000 ether, "1000 tokens lost");
        
        // Prove token disappearance
        uint256 buggyAccounted = buggyTotal + buggyFee;
        uint256 correctAccounted = correctTotal + correctFee;
        
        assertEq(correctAccounted, totalInput, "Correct: conserved");
        assertLt(buggyAccounted, totalInput, "Buggy: disappeared");
        assertEq(totalInput - buggyAccounted, 1_000 ether, "1000 unaccounted");
    }
}

```

Run the test:

```bash
forge test --match-test test_CumulativeFeeBug_POC -vv 
```

## Impact

- Users lose tokens beyond the intended fee
- Loss increases with number of flows in TVS
- Affects every split and merge operation

## Recommended Mitigation

Calculate and use the individual fee for each flow, not the cumulative total:

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
    public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {

    newAmounts = new uint256[](length);

    for (uint256 i; i < length;) {
        // Calculate individual fee for this flow
        uint256 individualFee = calculateFeeAmount(feeRate, amounts[i]);

        // Accumulate total fee (for treasury)
        feeAmount += individualFee;

        // Subtract only the individual fee, not cumulative
        newAmounts[i] = amounts[i] - individualFee;

        unchecked { ++i; }
    }
}
```

  