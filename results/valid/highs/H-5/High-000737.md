# [000737] Incorrect Fee Logic in FeesManager::calculateFeeAndNewAmountForOneTVS Causes Arithmetic Underflow and Loss of Funds
  
  ### Summary

The `FeesManager` contract contains a logic error in how fees are deducted during a `mergeTVS` operation. The function `calculateFeeAndNewAmountForOneTVS` calculates a cumulative running total of fees, but subtracts this total from each individual vesting flow.
This results in excessive fee deductions for all items after the first one. In most realistic scenarios (where subsequent vesting amounts are smaller than the cumulative fee of previous amounts), this causes an arithmetic underflow, reverting the transaction (DoS). In scenarios where it does not revert, it results in a significant, permanent loss of user funds.

### Root Cause

The error exists in `FeesManager.sol` lines within the `calculateFeeAndNewAmountForOneTVS` function:
```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;  //@> This subtracts the total cumulative fee from the current item
        }
    }
```

### Internal Pre-conditions

1. A user attempts to merge a TVS containing 2 or more flows.

### External Pre-conditions

_No response_

### Attack Path

Consider a user merging two vesting schedules (TVS) with a 10% Merge Fee (1000 Basis Points):
* Flow A: 1,000 Tokens
* Flow B: 50 Tokens

1. Iteration 0 (Flow A):
* currentFee = 1,000 * 10% = 100
* feeAmount (Total) = 0 + 100 = 100
* newAmounts[0] = 1,000 - 100 = 900 (Correct)

2. Iteration 1 (Flow B):currentFee = 50 * 10% = 5
 * feeAmount (Total) = 100 + 5 = 105
 * newAmounts[1] = 50 - 105

Result: Arithmetic Underflow (Revert). The transaction fails because the contract attempts to subtract the cumulative fee (105) from the small second amount (50).

Scenario Two (Loss Of Funds)
If Flow B had been larger (e.g., 200 tokens), it would not revert but would burn funds.
* newAmounts[1] = 200 - 105 = 95
* Expected: 200 - 5 = 195

Loss: The user loses 100 tokens (the fee from Flow A is incorrectly deducted again from Flow B).

### Impact

* Denial of Service: Users cannot merge vesting schedules if any subsequent flow amount is smaller than the sum of fees calculated for previous flows.
* Permanent Loss of Funds: If the transaction succeeds, the user is overcharged significantly, permanently losing assets equal to the sum of previous fees applied to subsequent entries.

### PoC
To demonstrate the severity, the Proof of Concept uses a test harness (`HarnessAlignerzVesting`) that overrides the `calculateFeeAndNewAmountForOneTVS` function. This override was necessary because the original `FeesManager.sol` contract contains two high severity blockers that prevent execution from ever reaching the math logic:
*  #6 
*  #7
 In `HarnessAlignerzVesting`, I applied minimal fixes for these two blockers:

* Explicitly allocated memory: `newAmounts = new uint256[](length)`;
* Added the loop increment: `for (uint256 i = 0; i < length; i++)`

Crucially, I preserved the flawed math logic exactly as it appears in the source code:
```solidity
feeAmount += currentFee;
newAmounts[i] = amounts[i] - feeAmount;
```
The following test was run and produced these logs: 
```bash
Ran 1 test for test/PoC.t.sol:FeesManagerMathLogicPoC
[PASS] testCumulativeFeeCausesUnderflow() (gas: 21113)
Logs:
  Testing Math Logic with:
  Flow 0: 1000 tokens
  Flow 1: 50 tokens
  Expected: Revert due to Arithmetic Underflow
  Test Passed: Math Logic caused Underflow as expected.

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.32ms (2.98ms CPU time)

Ran 1 test suite in 32.22ms (7.32ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
```bash
Ran 1 test for test/PoC.t.sol:FeesManagerMathLogicPoC
[PASS] testMathLogicStealsFunds() (gas: 19836)
Logs:
  Flow 0 Result: 90000000000000000000
  Flow 1 Result: 80000000000000000000
  Test Passed: User lost funds due to incorrect math.

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 8.74ms (1.21ms CPU time)

Ran 1 test suite in 68.25ms (8.74ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
The test:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {MockUSD} from "../src/MockUSD.sol";
// We don't need Upgrades library for direct deployment
// import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol"; 

// ----------------------------------------------------------------------------
// HARNESS CONTRACT
// ----------------------------------------------------------------------------
contract HarnessAlignerzVesting is AlignerzVesting {
    
    // Override the broken function to fix memory/loop bugs but KEEP the Math Bug.
    // Note: 'calculateFeeAndNewAmountForOneTVS' must be 'virtual' in FeesManager.sol
    function calculateFeeAndNewAmountForOneTVS(
        uint256 feeRate, 
        uint256[] memory amounts, 
        uint256 length
    ) public pure override returns (uint256 feeAmount, uint256[] memory newAmounts) {
        
        // 1. PATCH: Initialize memory (Fixes Panic 0x32)
        newAmounts = new uint256[](length); 

        // 2. PATCH: Add loop increment (Fixes Infinite Loop)
        for (uint256 i = 0; i < length; i++) { 
            // -------------------------------------------------------
            // VULNERABILITY REPLICATION (The Math Bug)
            // -------------------------------------------------------
            
            // Use inherited helper
            uint256 currentFee = calculateFeeAmount(feeRate, amounts[i]);
            
            feeAmount += currentFee;
            
            // BUG: Subtract the CUMULATIVE TOTAL from the CURRENT item.
            newAmounts[i] = amounts[i] - feeAmount; 
        }
    }
}

contract FeesManagerMathLogicPoC is Test {
    HarnessAlignerzVesting public vesting;
    AlignerzNFT public nft;
    Aligners26 public token;
    MockUSD public usdt;
    
    address public user;

    function setUp() public {
        user = makeAddr("user");
        token = new Aligners26("26Aligners", "A26Z");
        usdt = new MockUSD();
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");
        
        // FIX: Deploy Harness directly to avoid 'vm.readFile' artifact errors.
        // We don't need the Proxy verification logic for testing internal math functions.
        vesting = new HarnessAlignerzVesting();
        
        // Manually initialize since we aren't using the Proxy library
        vesting.initialize(address(nft));
    }
    
    function testCumulativeFeeCausesUnderflow() public {
        // Scenario: Merging two flows with 10% fee.
        // Flow 0: 1000 tokens. Fee = 100.
        // Flow 1: 50 tokens.   Fee = 5.
        
        // Math Bug Execution:
        // Iteration 0: feeAmount = 100. newAmount = 1000 - 100 = 900. (OK)
        // Iteration 1: feeAmount = 105. newAmount = 50 - 105. (UNDERFLOW)

        uint256[] memory amounts = new uint256[](2);
        amounts[0] = 1000 ether;
        amounts[1] = 50 ether; 
        
        uint256 feeRate = 1000; // 10% (1000 basis points)

        console.log("Testing Math Logic with:");
        console.log("Flow 0: 1000 tokens");
        console.log("Flow 1: 50 tokens");
        console.log("Expected: Revert due to Arithmetic Underflow");

        // We expect the transaction to crash with Panic 0x11 (Arithmetic Underflow)
        vm.expectRevert(stdError.arithmeticError);
        
        vesting.calculateFeeAndNewAmountForOneTVS(feeRate, amounts, 2);
        
        console.log("Test Passed: Math Logic caused Underflow as expected.");
    }

    function testMathLogicStealsFunds() public {
        // Scenario: Merging flows where underflow DOESN'T happen, but funds are lost.
        // Flow 0: 100 tokens. Fee = 10.
        // Flow 1: 100 tokens. Fee = 10.
        
        // Bug Execution:
        // Iteration 0: feeAmount = 10. newAmount = 90.
        // Iteration 1: feeAmount = 20. newAmount = 100 - 20 = 80. (Should be 90)
        
        uint256[] memory amounts = new uint256[](2);
        amounts[0] = 100 ether;
        amounts[1] = 100 ether;
        uint256 feeRate = 1000; // 10%

        (, uint256[] memory newAmounts) = vesting.calculateFeeAndNewAmountForOneTVS(feeRate, amounts, 2);

        console.log("Flow 0 Result:", newAmounts[0]);
        console.log("Flow 1 Result:", newAmounts[1]);

        assertEq(newAmounts[0], 90 ether, "Flow 0 correct");
        
        // Proves Flow 1 was charged double the fee (10 from Flow 0 + 10 from Flow 1)
        assertEq(newAmounts[1], 80 ether, "Flow 1 lost extra funds due to cumulative subtraction");
        
        console.log("Test Passed: User lost funds due to incorrect math.");
    }
}
```

### Mitigation

Modify the logic to subtract only the current fee from the current amount, while keeping the running total for the return value.


  