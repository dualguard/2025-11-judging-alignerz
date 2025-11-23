# [000070] Infinite Loop DoS in calculateFeeAndNewAmountForOneTVS()
  
  ## Summary

The [`calculateFeeAndNewAmountForOneTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169) function contains an infinite loop because the loop counter `i` is never incremented. This causes transactions to consume all available gas (Out of Gas) and revert. As a result, the [`splitTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1069) and [`mergeTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1013) functions are completely unusable.

Note: This bug is currently masked by an array initialization bug, but will cause infinite loop once that bug is fixed.

See separate finding: Uninitialized Array in calculateFeeAndNewAmountForOneTVS() Causes Complete DoS

## Vulnerability Details

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
    public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {

    // The loop initializes 'i' to 0, but the 3rd statement (iteration) is empty
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;

        // missing i++
        // 'i' remains 0 forever, so 'i < length' is always true
    }
}
```

### Root Cause

The loop counter `i` is never incremented. In Solidity `for` loops, when the iteration statement in the header is omitted (as seen here: `for (...; ...; )`), the counter must be manually incremented inside the loop body. Since this is missing, `i` remains 0 forever, causing `i < length` to always evaluate to true, resulting in an infinite loop.

## Impact

Broken Core Functionality:

- The split and merge features are core components of the protocol's TVS (Tokenized Vesting Schedule) utility
- These features are prominently advertised in the whitepaper as key innovations
- This bug renders them 100% non-functional

Gas Loss:

- Users attempting to interact with these functions will have their transactions fail only after consuming the maximum amount of gas allowed
- Results in wasted funds for every attempted merge or split operation

This functionality is completely unavailable due to this bug.

## Proof of Concept

Add this test in /test folder

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";

contract InfiniteLoopTest is Test {
    uint256 constant BASIS_POINT = 10_000;

    function calculateFeeAmount(uint256 feeRate, uint256 amount) public pure returns(uint256 feeAmount) {
        feeAmount = amount * feeRate / BASIS_POINT;
    }

    // missing i++ causes infinite loop
    function buggy_calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) 
        public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint256[](length); // we also had to add this line
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
            // missing i++ causes infinite loop
        }
    }

    function fixed_calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) 
        public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
            unchecked { ++i; } // fix: Increment i
        }
    }

    // Test 1: succeeds
    function test_FixedVersion_Succeeds() public pure {
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 1000 ether;
        amounts[1] = 2000 ether;
        amounts[2] = 3000 ether;

        // This completes successfully
        (, uint256[] memory newAmounts) = fixed_calculateFeeAndNewAmountForOneTVS(100, amounts, 3);
        
        assertEq(newAmounts.length, 3);
    }

    // Test 2: runs out of gas
    function test_BuggyVersion_OutOfGas() public {
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 1000 ether;

        vm.expectRevert();
        this.buggy_calculateFeeAndNewAmountForOneTVS{gas: 500_000}(100, amounts, 1);
    }
}
```

## Recommended Mitigation

Add the increment operator at the end of the loop body. Using an `unchecked` block is recommended for gas optimization, as `i` will not overflow `length`. Also initialize the `newAmounts` array.

  