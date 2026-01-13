# [000119] 🔴 high: FeesManager will Over-Deduct Tokens for Vesting Flows leading to Loss for Vesting Recipients
  
  ### Summary

​A logic error in fee calculation will cause a critical financial loss for vesting recipients as the FeesManager will incorrectly subtract the cumulative total fee from every subsequent flow's amount.

### Root Cause

​In [protocol/src/contracts/vesting/feesManager/FeesManager.sol#L182-L184](https://github.com/dualguard/2025-11-alignerz/blob/ad09cb2762a57ae927e83c4fc5c4db98d810dc86/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L182-L184), the function calculateFeeAndNewAmountForOneTVS reuses the feeAmount variable for two purposes:
1. ​Accumulating the total fee (feeAmount += currentFee;).
2. ​Subtracting the fee from the current flow's amount (newAmounts[i] = amounts[i] - feeAmount;).

​Because the fee is subtracted after being accumulated, the resulting amount for any flow i > 0 has all fees from Flow 0 up to Flow i subtracted from it, instead of just the fee for Flow i.


### Internal Pre-conditions

1. FeesManager.sol needs to be initialized with mergeFeeRate or splitFeeRate to be greater than 0.

​2. The input array amounts must contain at least two vesting flows.


### External Pre-conditions

​None. The vulnerability is internal to the contract logic and requires no external market manipulation.


### Attack Path

​This is a vulnerability path, not an attack path, as the incorrect calculation occurs automatically upon calling the affected functions:
​1. A vesting recipient calls an operation that modifies vesting streams, such as MergeTVS or SplitTVS.
​2. The operation calls [calculateFeeAndNewAmountForOneTVS](https://github.com/dualguard/2025-11-alignerz/blob/ad09cb2762a57ae927e83c4fc5c4db98d810dc86/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L182-L184).
3. ​Inside the loop, for the first flow (i=0), the contract correctly subtracts Fee 0.
​4. For the second flow (i=1), the contract calculates Flow 1's fee (F_1).
​5. The variable feeAmount now holds F_0 + F_1.
​6. The contract calculates the new amount: {New Amount}_1 = {Initial Amount}_1 - (F_0 + F_1).
​7. The calculated resulting amount is substantially lower than intended.

### Impact

​The vesting recipients suffer a loss of tokens equal to the cumulative sum of all preceding flow fees for every subsequent flow. The loss value is proportional to the total fee collected up to that point.
​Example Loss: With two flows of 100,000 and 50,000 units and a 5\% fee:
​1. Expected Net Flow 1: 47,500 units.
​2. Actual Net Flow 1: 42,500 units.
​3. Loss: 5,000 units (equal to the fee from Flow 0).


### PoC

​The following Foundry test demonstrates the failure where Flow 1 is calculated as 42,500 units instead of the correct 47,500 units.
​Test File Reference: protocol/test/AlignerzVestingProtocolTest.t.sol

```javascript
  
    function test_calculateFee_buggy_vs_expected() public {
        // --- 1. SETUP TEST DATA ---
        uint256 feeRate = 500; // 500 BP = 5.00%
        uint256 BASIS_POINT = 10_000;

        // Initialize the amounts array with sample flows
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 100_000 ether; // 100,000 tokens
        amounts[1] = 50_000 ether; // 50,000 tokens
        amounts[2] = 20_000 ether; // 20,000 tokens
        uint256 length = amounts.length;

        // --- 2. CALL THE BUGGY CONTRACT FUNCTION ---
        (uint256 actualFeeAmount, uint256[] memory actualAmounts) = feesManager
            .calculateFeeAndNewAmountForOneTVS(feeRate, amounts, length);

        // --- 3. BUILD CORRECT EXPECTED AMOUNTS ---
        uint256[] memory expectedAmounts = new uint256[](length);
        uint256[] memory fees = new uint256[](length);
        uint256 totalExpectedFee = 0;

        for (uint256 i = 0; i < length; i++) {
            // Calculate FEE correctly for this flow (per-flow fee)
            fees[i] = (amounts[i] * feeRate) / BASIS_POINT;
            totalExpectedFee += fees[i];

            // Calculate NEW AMOUNT correctly for this flow
            expectedAmounts[i] = amounts[i] - fees[i];
        }

        // --- 4. ASSERTIONS ---
        // Assert the total fee collected is correct
        assertEq(
            actualFeeAmount,
            totalExpectedFee,
            "Total Fee collected is wrong"
        );

        // Assert the corrected amount against the buggy amount (This will FAIL and confirm the bug)
        for (uint256 i; i < length; i++) {
            // The test will fail here, proving the contract logic is wrong
            assertEq(
                actualAmounts[i],
                expectedAmounts[i],
                string.concat("Flow ", i.toString(), " amount is wrong")
            );
        }
    }
```
Technical Notes on PoC Setup (Crucial for Reproduction)
​To successfully execute this proof-of-concept, the following modifications to the testing environment were required, which highlights issues with the base contract's testability:
​Dependency Issue: Since the functions in the base FeesManager.sol were not declared as virtual, a FeesManagerMock contract was necessary.
​Secondary Bug Fix (Array Initialization): The original function would crash immediately (panic 0x32) because the newAmounts array was not initialized inside the pure function. The mock was engineered to fix this secondary bug to allow the primary logic error to be tested.
​FeesManagerMock Implementation
​This is the exact contract used for the PoC, featuring the required array initialization fix while preserving the primary logic bug for testing:

```javascript
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import {FeesManager} from "src/contracts/vesting/feesManager/FeesManager.sol";
import {console} from "forge-std/console.sol"; // Useful for debugging

contract FeesManagerMock is FeesManager {
    // NOTE: You must include the calculateFeeAmount utility function
    // or ensure it's inherited and callable, as the main function depends on it.

    function calculateFeeAmount(
        uint256 feeRate,
        uint256 amount
    ) public pure override returns (uint256 feeAmount) {
        feeAmount = (amount * feeRate) / BASIS_POINT;
    }

    function calculateFeeAndNewAmountForOneTVS(
        uint256 feeRate,
        uint256[] memory amounts,
        uint256 length
    )
        public
        pure
        override
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
        // 1. FIX THE PANIC: Initialize the return array size
        newAmounts = new uint256[](length);

        // 2. The original buggy logic follows:
        for (uint256 i; i < length; ) {
            // This is the buggy part you want to prove!
            uint256 currentFee = calculateFeeAmount(feeRate, amounts[i]);
            feeAmount += currentFee;

            // BUG: Subtracts cumulative fee
            newAmounts[i] = amounts[i] - feeAmount;

            unchecked {
                ++i;
            }
        }
    }

    function initializeMock(
        uint256 _mergeFeeRate,
        uint256 _splitFeeRate
    ) public {
        mergeFeeRate = _mergeFeeRate;
        splitFeeRate = _splitFeeRate;
    }
}
```

### Mitigation

Ran 1 test for test/AlignerzVestingProtocolTest.t.sol:AlignerzVestingProtocolTest
[FAIL: Flow 1 amount is wrong: 42500000000000000000000 != 47500000000000000000000] test_calculateFee_buggy_vs_expected() 
// ...

## Mitigation

The fee accumulation variable must be separated from the fee subtraction variable. A local variable must be introduced to hold the fee for the current flow, and this local variable should be used exclusively for the subtraction step.

### Fixed Contract Code

The fix should be applied to [protocol/src/contracts/vesting/feesManager/FeesManager.sol](https://github.com/dualguard/2025-11-alignerz/blob/ad09cb2762a57ae927e83c4fc5c4db98d810dc86/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L182-L184).

```solidity
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 totalFeeAmount, uint256[] memory newAmounts) {
    
    // CRITICAL FIX: Ensure return array is initialized (Required to prevent Panic 0x32)
    newAmounts = new uint256[](length);
    totalFeeAmount = 0; // Initialize total fee accumulator
    
    for (uint256 i; i < length; ) {
        
        // 1. Calculate fee for the CURRENT flow only
        uint256 currentFlowFee = calculateFeeAmount(feeRate, amounts[i]);
        
        // 2. Accumulate the total fee for the return value (totalFeeAmount is cumulative)
        totalFeeAmount += currentFlowFee;
        
        // 3. FIX: Subtract ONLY the current flow's fee (currentFlowFee)
        newAmounts[i] = amounts[i] - currentFlowFee; 
        
        unchecked {
            ++i;
        }
    }
}
  