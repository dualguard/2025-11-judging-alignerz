# [000943] Missing Loop Counter Increment Causes Out of Gas Failures
  
  ### Summary

The `calculateFeeAndNewAmountForOneTVS` function contains a for loop that never increments its counter variable, creating an infinite loop. Even if the uninitialized array bug were fixed, any merge or split operation would run indefinitely until the transaction runs out of gas and reverts.

### Root Cause

The function iterates over multiple flows to calculate fees, but the loop counter is never incremented:
```solidity
FeesManager.sol
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate, 
    uint256[] memory amounts, 
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) { //@audit loop counter never increments
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
        // Missing: ++i or unchecked { ++i; }
    }
}
```
Without incrementing i, the loop condition i < length remains forever true (0 < 2), causing the loop to execute indefinitely on the same iteration.

> [https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169](url)


### Internal Pre-conditions

none

### External Pre-conditions

none

### Attack Path

none

### Impact

Guaranteed transaction failure with gas loss:
All merge operations fail with out of gas error
All split operations fail with out of gas error
Users lose gas fees on every failed attempt
No merge or split can succeed regardless of input parameters

### PoC

(Walkthrough)
Scenario: Bob attempts to merge 2 NFTs with amounts [5000, 1000].
Execution flow:
```text
Step 1: Loop starts with i = 0, length = 2
Step 2: Condition check: 0 < 2 → TRUE, enter loop
Step 3: Calculate fee for amounts[0]
Step 4: Assign newAmounts[0]
Step 5: Loop body complete, return to condition check
Step 6: Condition check: 0 < 2 → STILL TRUE (i never changed)
Step 7: Enter loop again with same i = 0
Step 8: Calculate fee for amounts[0] AGAIN
Step 9: Assign newAmounts[0] AGAIN
... (repeats indefinitely)
```
The transaction continues executing the same iteration until it exhausts the block gas limit and reverts with out of gas error. Bob's merge fails, and he loses the gas spent on the failed transaction.

### Mitigation

Add the missing loop counter increment.
  