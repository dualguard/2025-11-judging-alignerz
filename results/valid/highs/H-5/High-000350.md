# [000350] [H-1] Incorrect fee deduction logic in `FeesManager.sol::calculateFeeAndNewAmountForOneTVS#69` will Miscalculate all Token Vesting Splits (TVS)
  
  ### Summary

Incorrect cumulative fee subtraction in `FeesManager.sol::calculateFeeAndNewAmountForOneTVS#69` will cause a miscalculation of vesting distributions  for all recipients as the function will allocate incorrect amounts.

### Root Cause

In the [`FeesManager.sol::calculateFeeAndNewAmountForOneTVS#169`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169) function, the fee is accumulated across all iterations but incorrectly subtracted from each individual amount, instead of subtracting only the individual fee calculated for that specific amount.:

```solidity
    function calculateFeeAndNewAmountForOneTVSA(...) public pure returns (...) {
        for (uint256 i; i < length; i++) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]); // Accumulates
            // @audit-issue: subtracts accumulated total
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

### Internal Pre-conditions


1. The vesting system invokes `FeesManager.sol::calculateFeeAndNewAmountForOneTVS` during TVS splitting.
2. The `amounts` array contains more than one split value.
3. The fee mechanism is enabled and relies on this function's output.

### External Pre-conditions

_No response_

### Attack Path

1. A vesting split operation like `AlignerzVesting.sol::splitTVS#1056` triggers the execution of `FeesManager.sol::calculateFeeAndNewAmountForOneTVS`.
2. Cumulative fee subtraction causes (Example from the AlignerZ Whitepaper 2.6):
    - TVS-1 = 70 - f₁
    - TVS-2 = 20 - (f₁ + f₂)
    - TVS-3 = 10 - (f₁ + f₂ + f₃)
4. This results in:
    - Incorrect distribution of vesting amounts
    - Over charging of fees on later splits
    - Potential underflow
5. Total vesting amounts become inconsistent, violating the intended design (fee applied once at TVS level).

### Impact

This causes a direct loss of users funds. TVS holders suffer an approximate loss of 0.5% to 10% of their tokens depending on the number of splits performed, significantly exceeding the documented 0.5% fee:

### PoC
I also assume the previous two issues have been rectifies, If not, I included it below for reference

```solidity
        function calculateFeeAndNewAmountForOneTVS(
            uint256 feeRate,
            uint256[] memory amounts,
            uint256 length
    ) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
            newAmounts = new uint256[](length);
            for (uint256 i; i < length; i++) {
                feeAmount += calculateFeeAmount(feeRate, amounts[i]);
                newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```


For this PoC, copy the code and create a new test file inside test folder
I wrote two test cases for this to show how the fees accumulate as the number of splits increases

```solidity

    // SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {FeesManager} from "../src/contracts/vesting/feesManager/FeesManager.sol";
import {Test, console} from "forge-std/Test.sol";

// FeesManager is an abstract contract
contract Manager is FeesManager {

}

contract FeeManagerTest is Test {
    Manager manager;
    function setUp() public {
        manager = new Manager();
    }

    function testFeeCalculation() public {        
        // User splits TVS into 3 parts: 70%, 20%, 10%
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 7000e18;
        amounts[1] = 2000e18;
        amounts[2] = 1000e18;
        // Total: 10000 tokens
        
        uint256 feeRate = 50;  // 0.5%
        
        (uint256 totalFee, uint256[] memory newAmounts) = manager.calculateFeeAndNewAmountForOneTVS(
            feeRate,
            amounts,
            3
        );

        console.log("Input: 3 splits of (70%, 20%, 10%) tokens each = 1200 tokens total");
        console.log("Fee rate: 0.5%");
        console.log("");
        
        console.log("Results:");
        uint256 sum = 0;
        for (uint i = 0; i < 3; i++) {
            console.log("newAmount[%d]:", i, newAmounts[i] / 1e18, "tokens");
            sum += newAmounts[i];
        }
        
        uint256 totalAfterFee = 10000e18 - ((10000e18 * 50) / 10000);
        console.log("Expected total after fee:", totalAfterFee / 1e18, "tokens");
        console.log("Actual total returned:", sum / 1e18, "tokens");
        console.log("Total loss:", (totalAfterFee - sum) / 1e18, "tokens");
    }

    function testForMoeToken() public view {        
        uint256[] memory amounts = new uint256[](10);
        for (uint i = 0; i < 10; i++) {
            amounts[i] = 100e18;  // 100 tokens each
        }
        
        uint256 feeRate = 50;  // 0.5%
        
        (uint256 totalFee, uint256[] memory newAmounts) = manager.calculateFeeAndNewAmountForOneTVS(
            feeRate,
            amounts,
            10
        );
        
        console.log("Input: 10 splits of 100 tokens each = 1000 tokens total");
        console.log("Fee rate: 0.5%");
        console.log("");
        
        console.log("Results:");
        uint256 sum = 0;
        for (uint i = 0; i < 10; i++) {
            console.log("Split[%d]:", i, newAmounts[i] / 1e18, "tokens");
            sum += newAmounts[i];
        }
        
        uint256 totalAfterFee = 1000e18 - ((1000e18 * 50) / 10000);
        console.log("Expected total after fee:", totalAfterFee / 1e18, "tokens");
        console.log("Actual total returned:", sum / 1e18, "tokens");
        console.log("Total loss:", (totalAfterFee - sum) / 1e18, "tokens");
        console.log(totalAfterFee);
    }
}

```

### Mitigation

For this code, I already included the previously mentioned bugs
So I'm only pointing out the only issue for this particular issue

```diff
        function calculateFeeAndNewAmountForOneTVSA(...) public pure returns (...) {
        newAmounts = new uint256[](length);
        
        for (uint256 i; i < length; i++) {
-           feeAmount += calculateFeeAmount(feeRate, amounts[i]);
-           newAmounts[i] = amounts[i] - feeAmount;

            // Calculate fee for this amount only
+          uint256 individualFee = calculateFeeAmount(feeRate, amounts[i]);
            
            // Accumulate total fee
+          feeAmount += individualFee;
            
            // Subtract only this amount's fee (not accumulated)
+            newAmounts[i] = amounts[i] - individualFee;

        }
    }
```
  