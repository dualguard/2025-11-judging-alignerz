# [000338] Users will lose funds due to incorrect cumulative fee calculation
  
  ### Summary

A mathematical error in the fee calculation function will cause incorrect fee deductions for users splitting or merging TVS with multiple flows, as the function subtracts cumulative fees from each amount instead of subtracting only the fee for that specific amount.

### Root Cause

In [FeesManager.sol:172](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L172) the function `calculateFeeAndNewAmountForOneTVS` subtracts the cumulative `feeAmount` from each `amounts[i]` instead of subtracting only the fee for that specific amount.

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;  //  //@audit-issue Subtracts cumulative fee instead of fee!
    }
}
```

### Internal Pre-conditions

1. User must have a TVS with multiple flows (amounts array length > 1)
2. User must call `splitTVS()` or `mergeTVS()` which uses `calculateFeeAndNewAmountForOneTVS()`
3. `splitFeeRate` or `mergeFeeRate` must be greater than 0

### External Pre-conditions

None

### Attack Path

1. User has a TVS with multiple flows, e.g., amounts = [1000, 2000, 3000] tokens
2. User calls `splitTVS()` or `mergeTVS()` with fee rate 50 BPS (0.5%)
3. Function calculates fees:
   - Iteration 0: feeAmount = 5, newAmounts[0] = 1000 - 5 = 995 (correct)
   - Iteration 1: feeAmount = 15, newAmounts[1] = 2000 - 15 = 1985 (incorrect, should be 1990)
   - Iteration 2: feeAmount = 30, newAmounts[2] = 3000 - 30 = 2970 (incorrect, should be 2985)
4. User receives less tokens than intended for later flows in the array

### Impact

Users splitting or merging TVS with multiple flows suffer incorrect fee deductions. Later flows in the array are charged fees from earlier flows. For example, with amounts [1000, 2000, 3000] and 0.5% fee rate, the user loses 20 tokens (5 from second flow + 15 from third flow) compared to the correct calculation.

Loss of funds => High Severity label.

### PoC

**PLEASE NOTE**: The `calculateFeeAndNewAmountForOneTVS` function is bugged initially and times out (DOS) due to the increment `i++` not being present (infinite loop). This POC fixes the function by adding the increment to prove another bug. The infinite loop DOS is described in another submission. Additionally, that same function didn't instantiate the return array `uint256[] memory newAmounts`'s size: therefore any attempt to access it (even `newAmounts[0]` will result in a `[FAIL: panic: array out-of-bounds access (0x32)]`. So this was also fixed to showcase the current bug

Run the coded POC:
```bash
forge test --match-path "test/CumulativeFeeBug.t.sol" -vv
```

The test demonstrates:
- Direct bug demonstration showing incorrect calculations
- Step-by-step trace of the bug
- Impact with different fee rates
- Correct implementation comparison

Screenshot of the execution:

<img width="930" height="787" alt="Image" src="https://github.com/user-attachments/assets/8e5f68e2-7a6b-4bef-8c52-2c6ab60398b0" />

<img width="577" height="641" alt="Image" src="https://github.com/user-attachments/assets/4d5118ad-58cf-4aff-b889-e313b56fca73" />

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import {Test, console} from "forge-std/Test.sol";
import {FeesManager} from "../src/contracts/vesting/feesManager/FeesManager.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";

/**
 * @title CumulativeFeeBug
 * @notice POC demonstrating the cumulative fee subtraction bug in calculateFeeAndNewAmountForOneTVS
 * 
 * THE BUG:
 * Line 172: newAmounts[i] = amounts[i] - feeAmount;
 * 
 * This subtracts the CUMULATIVE feeAmount from each amount, instead of subtracting only
 * the fee for that specific amount. This causes later array elements to be charged
 * fees from earlier elements.
 * 
 * EXAMPLE:
 * amounts = [1000, 2000, 3000], feeRate = 50 (0.5%)
 * 
 * Current (BUGGY) behavior:
 * - i=0: feeAmount = 5, newAmounts[0] = 1000 - 5 = 995 ✓
 * - i=1: feeAmount = 15, newAmounts[1] = 2000 - 15 = 1985 ✗ (should be 1990)
 * - i=2: feeAmount = 30, newAmounts[2] = 3000 - 30 = 2970 ✗ (should be 2985)
 * 
 * Correct behavior should be:
 * - i=0: fee = 5, newAmounts[0] = 1000 - 5 = 995 ✓
 * - i=1: fee = 10, newAmounts[1] = 2000 - 10 = 1990 ✓
 * - i=2: fee = 15, newAmounts[2] = 3000 - 15 = 2985 ✓
 * 
 * IMPACT:
 * Users splitting/merging TVS with multiple flows pay incorrect (higher) fees.
 * Later flows in the array are penalized with fees from earlier flows.
 */
contract CumulativeFeeBug is Test {
    // Create a test contract that exposes the function
    TestFeesManager public feesManager;
    
    function setUp() public {
        feesManager = new TestFeesManager();
    }
    
    /**
     * @notice Direct test of the buggy function
     */
    function test_CumulativeFeeBugDirect() public {
        uint256 feeRate = 50; // 0.5% = 50/10000
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 1000 * 1e18;
        amounts[1] = 2000 * 1e18;
        amounts[2] = 3000 * 1e18;
        
        console.log("=== TESTING BUGGY FUNCTION ===");
        console.log("Input amounts:");
        console.log("  amounts[0] =", amounts[0]);
        console.log("  amounts[1] =", amounts[1]);
        console.log("  amounts[2] =", amounts[2]);
        console.log("Fee rate: 50 BPS (0.5%)");
        console.log("");
        
        // Call the buggy function
        (uint256 totalFee, uint256[] memory newAmounts) = feesManager.calculateFeeAndNewAmountForOneTVS(
            feeRate,
            amounts,
            amounts.length
        );
        
        console.log("=== BUGGY OUTPUT ===");
        console.log("Total fee collected:", totalFee);
        console.log("New amounts after fee deduction:");
        console.log("  newAmounts[0] =", newAmounts[0]);
        console.log("  newAmounts[1] =", newAmounts[1]);
        console.log("  newAmounts[2] =", newAmounts[2]);
        console.log("");
        
        // Calculate what it SHOULD be
        console.log("=== CORRECT CALCULATION ===");
        uint256 expectedFee0 = amounts[0] * feeRate / 10000;
        uint256 expectedFee1 = amounts[1] * feeRate / 10000;
        uint256 expectedFee2 = amounts[2] * feeRate / 10000;
        uint256 expectedTotalFee = expectedFee0 + expectedFee1 + expectedFee2;
        
        uint256 correctAmount0 = amounts[0] - expectedFee0;
        uint256 correctAmount1 = amounts[1] - expectedFee1;
        uint256 correctAmount2 = amounts[2] - expectedFee2;
        
        console.log("Expected total fee:", expectedTotalFee);
        console.log("Expected new amounts:");
        console.log("  newAmounts[0] =", correctAmount0);
        console.log("  newAmounts[1] =", correctAmount1);
        console.log("  newAmounts[2] =", correctAmount2);
        console.log("");
        
        // Show the difference
        console.log("=== BUG DEMONSTRATION ===");
        console.log("Element 0:");
        console.log("  Buggy result:", newAmounts[0]);
        console.log("  Correct result:", correctAmount0);
        console.log("  Match:", newAmounts[0] == correctAmount0 ? "YES" : "NO");
        console.log("");
        
        console.log("Element 1:");
        console.log("  Buggy result:", newAmounts[1]);
        console.log("  Correct result:", correctAmount1);
        console.log("  Difference:", newAmounts[1] < correctAmount1 ? 
            vm.toString(correctAmount1 - newAmounts[1]) : "0");
        console.log("  Match:", newAmounts[1] == correctAmount1 ? "YES" : "NO");
        console.log("");
        
        console.log("Element 2:");
        console.log("  Buggy result:", newAmounts[2]);
        console.log("  Correct result:", correctAmount2);
        console.log("  Difference:", newAmounts[2] < correctAmount2 ? 
            vm.toString(correctAmount2 - newAmounts[2]) : "0");
        console.log("  Match:", newAmounts[2] == correctAmount2 ? "YES" : "NO");
        console.log("");
        
        // Assertions
        assertEq(newAmounts[0], correctAmount0, "First element should be correct");
        assertNotEq(newAmounts[1], correctAmount1, "Second element should be WRONG (bug)");
        assertNotEq(newAmounts[2], correctAmount2, "Third element should be WRONG (bug)");
        
        // Show user loses money
        uint256 userLoss = (correctAmount1 - newAmounts[1]) + (correctAmount2 - newAmounts[2]);
        console.log("=== USER LOSS ===");
        console.log("Total amount user loses due to bug:", userLoss);
        console.log("User pays extra fees:", userLoss);
        
        assertGt(userLoss, 0, "User should lose money due to bug");
    }
    
    /**
     * @notice Trace through the bug step by step
     */
    function test_CumulativeFeeBugStepByStep() public {
        uint256 feeRate = 50; // 0.5%
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 1000 * 1e18;
        amounts[1] = 2000 * 1e18;
        amounts[2] = 3000 * 1e18;
        
        console.log("=== STEP-BY-STEP TRACE ===");
        console.log("Initial amounts: [1000, 2000, 3000] tokens");
        console.log("Fee rate: 50 BPS (0.5%)");
        console.log("");
        
        // Simulate the buggy loop
        uint256 feeAmount = 0;
        uint256[] memory newAmounts = new uint256[](3);
        
        // Iteration 0
        uint256 fee0 = amounts[0] * feeRate / 10000; // 5 tokens
        feeAmount += fee0; // feeAmount = 5
        newAmounts[0] = amounts[0] - feeAmount; // 1000 - 5 = 995
        console.log("Iteration 0:");
        console.log("  Fee for amounts[0]:", fee0);
        console.log("  Cumulative feeAmount:", feeAmount);
        console.log("  newAmounts[0] = amounts[0] - feeAmount =", newAmounts[0]);
        console.log("  [CORRECT] First element is correct");
        console.log("");
        
        // Iteration 1
        uint256 fee1 = amounts[1] * feeRate / 10000; // 10 tokens
        feeAmount += fee1; // feeAmount = 15
        newAmounts[1] = amounts[1] - feeAmount; // 2000 - 15 = 1985
        console.log("Iteration 1:");
        console.log("  Fee for amounts[1]:", fee1);
        console.log("  Cumulative feeAmount:", feeAmount);
        console.log("  newAmounts[1] = amounts[1] - feeAmount =", newAmounts[1]);
        console.log("  [BUG] Should be: amounts[1] - fee1 =", amounts[1] - fee1);
        console.log("  User loses:", (amounts[1] - fee1) - newAmounts[1], "tokens");
        console.log("");
        
        // Iteration 2
        uint256 fee2 = amounts[2] * feeRate / 10000; // 15 tokens
        feeAmount += fee2; // feeAmount = 30
        newAmounts[2] = amounts[2] - feeAmount; // 3000 - 30 = 2970
        console.log("Iteration 2:");
        console.log("  Fee for amounts[2]:", fee2);
        console.log("  Cumulative feeAmount:", feeAmount);
        console.log("  newAmounts[2] = amounts[2] - feeAmount =", newAmounts[2]);
        console.log("  [BUG] Should be: amounts[2] - fee2 =", amounts[2] - fee2);
        console.log("  User loses:", (amounts[2] - fee2) - newAmounts[2], "tokens");
        console.log("");
        
        console.log("=== SUMMARY ===");
        console.log("Total fee collected:", feeAmount);
        console.log("Total user loss:", ((amounts[1] - fee1) - newAmounts[1]) + ((amounts[2] - fee2) - newAmounts[2]));
        console.log("Later elements are charged fees from earlier elements!");
        
        // Verify the bug
        assertEq(newAmounts[0], 995 * 1e18, "First element correct");
        assertEq(newAmounts[1], 1985 * 1e18, "Second element has bug");
        assertEq(newAmounts[2], 2970 * 1e18, "Third element has bug");
        assertEq(newAmounts[1], 1990 * 1e18 - 5 * 1e18, "Second element lost 5 tokens from first");
        assertEq(newAmounts[2], 2985 * 1e18 - 15 * 1e18, "Third element lost 15 tokens from first two");
    }
    
    /**
     * @notice Show impact with different fee rates
     */
    function test_CumulativeFeeBugWithDifferentRates() public {
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 1000 * 1e18;
        amounts[1] = 2000 * 1e18;
        amounts[2] = 3000 * 1e18;
        
        console.log("=== IMPACT WITH DIFFERENT FEE RATES ===");
        console.log("Amounts: [1000, 2000, 3000] tokens");
        console.log("");
        
        // Test with 1% fee (100 BPS)
        uint256 feeRate1 = 100;
        (uint256 fee1, uint256[] memory newAmounts1) = feesManager.calculateFeeAndNewAmountForOneTVS(
            feeRate1, amounts, amounts.length
        );
        uint256 loss1 = (amounts[1] - (amounts[1] * feeRate1 / 10000)) - newAmounts1[1] +
                       (amounts[2] - (amounts[2] * feeRate1 / 10000)) - newAmounts1[2];
        console.log("Fee rate: 100 BPS (1%)");
        console.log("  User loss:", loss1);
        console.log("");
        
        // Test with 2% fee (200 BPS) - max allowed
        uint256 feeRate2 = 200;
        (uint256 fee2, uint256[] memory newAmounts2) = feesManager.calculateFeeAndNewAmountForOneTVS(
            feeRate2, amounts, amounts.length
        );
        uint256 loss2 = (amounts[1] - (amounts[1] * feeRate2 / 10000)) - newAmounts2[1] +
                       (amounts[2] - (amounts[2] * feeRate2 / 10000)) - newAmounts2[2];
        console.log("Fee rate: 200 BPS (2%)");
        console.log("  User loss:", loss2);
        console.log("");
        
        // Test with 0.5% fee (50 BPS) - whitepaper rate
        uint256 feeRate3 = 50;
        (uint256 fee3, uint256[] memory newAmounts3) = feesManager.calculateFeeAndNewAmountForOneTVS(
            feeRate3, amounts, amounts.length
        );
        uint256 loss3 = (amounts[1] - (amounts[1] * feeRate3 / 10000)) - newAmounts3[1] +
                       (amounts[2] - (amounts[2] * feeRate3 / 10000)) - newAmounts3[2];
        console.log("Fee rate: 50 BPS (0.5%) - Whitepaper rate");
        console.log("  User loss:", loss3);
        console.log("");
        
        console.log("=== CONCLUSION ===");
        console.log("Higher fee rates = Higher user loss");
        console.log("More flows = More user loss");
        console.log("Later flows always lose more than earlier flows");
        
        assertGt(loss2, loss1, "Higher fee rate causes more loss");
        assertGt(loss1, loss3, "Higher fee rate causes more loss");
    }
    
    /**
     * @notice Show the fix
     */
    function test_CorrectImplementation() public {
        uint256 feeRate = 50;
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 1000 * 1e18;
        amounts[1] = 2000 * 1e18;
        amounts[2] = 3000 * 1e18;
        
        console.log("=== CORRECT IMPLEMENTATION ===");
        
        // Correct implementation
        uint256 totalFee = 0;
        uint256[] memory correctAmounts = new uint256[](3);
        
        for (uint256 i; i < amounts.length;) {
            uint256 fee = amounts[i] * feeRate / 10000; // Calculate fee for THIS amount only
            totalFee += fee;
            correctAmounts[i] = amounts[i] - fee; // Subtract only THIS amount's fee
            unchecked { ++i; }
        }
        
        console.log("Correct total fee:", totalFee);
        console.log("Correct new amounts:");
        console.log("  correctAmounts[0] =", correctAmounts[0]);
        console.log("  correctAmounts[1] =", correctAmounts[1]);
        console.log("  correctAmounts[2] =", correctAmounts[2]);
        console.log("");
        console.log("Each amount only has its own fee deducted!");
        
        // Compare with buggy version
        (uint256 buggyFee, uint256[] memory buggyAmounts) = feesManager.calculateFeeAndNewAmountForOneTVS(
            feeRate, amounts, amounts.length
        );
        
        assertEq(totalFee, buggyFee, "Total fee is same (correct)");
        assertEq(correctAmounts[0], buggyAmounts[0], "First element same");
        assertNotEq(correctAmounts[1], buggyAmounts[1], "Second element different");
        assertNotEq(correctAmounts[2], buggyAmounts[2], "Third element different");
    }
}

/**
 * @notice Test contract that exposes the internal function
 */
contract TestFeesManager {
    uint256 constant public BASIS_POINT = 10_000;
    
    function calculateFeeAndNewAmountForOneTVS(
        uint256 feeRate, 
        uint256[] memory amounts, 
        uint256 length
    ) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount; // BUG: Subtracts cumulative fee
            unchecked { ++i; }
        }
    }
    
    function calculateFeeAmount(uint256 feeRate, uint256 amount) public pure returns(uint256 feeAmount) {
        feeAmount = amount * feeRate / BASIS_POINT;
    }
}
```

### Mitigation

Change line 172 in FeesManager.sol to subtract only the fee for the current amount:

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    newAmounts = new uint256[](length);
    for (uint256 i; i < length;) {
        uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
        feeAmount += fee;
        newAmounts[i] = amounts[i] - fee;  // @audit Fix: Subtract only this amount's fee
        unchecked { ++i; }
    }
}
```
  