# [000385] Cumulative Fee Deduction Vulnerability in FeesManager:calculateFeeAndNewAmountForOneTVS()
  
  ### Summary

The `calculateFeeAndNewAmountForOneTVS` function incorrectly deducts cumulative fees instead of individual fees, causing exponentially increasing charges for users.

### Root Cause
Subtracting cumulative fee instead of individual fee for each amount.
[FeesManager.sol:calculateFeeAndNewAmountForOneTVS() Line 169 -174 ](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174)
```solidity
// VULNERABLE CODE
for (uint256 i; i < length;) {
    feeAmount += calculateFeeAmount(feeRate, amounts[i]); // Accumulates total
    newAmounts[i] = amounts[i] - feeAmount;              // Deducts total instead of individual
}
```


### Internal Pre-conditions

User have TVS with multiple amounts.

### Attack Path

1. User initiates TVS operation with multiple amounts
2. First amount: `newAmounts[0] = amounts[0] - fee0` ✓
3. Second amount: `newAmounts[1] = amounts[1] - (fee0 + fee1)` ✗
4. Third amount: `newAmounts[2] = amounts[2] - (fee0 + fee1 + fee2)` ✗

### Impact

**Critical Cascading Effect:**
- **Fund Depletion:** Excessive fee deductions deplete the contract's token reserves
- **Last Users Cannot Claim:** Later users will be unable to claim their vested tokens due to insufficient contract balance
- **Protocol Insolvency:** Contract becomes insolvent as it owes more tokens than it holds

### PoC

**Test Scenario:**

```solidity
// Input parameters
uint256[] memory amounts = [1000, 1000, 1000, 1000, 1000]; // 5 users, 1000 tokens each
uint256 feeRate = 100; // 1% fee (100 basis points)
uint256 length = 5;

// Expected behavior (correct):
// Individual fees: [10, 10, 10, 10, 10] = 50 total
// New amounts: [990, 990, 990, 990, 990]
// Contract balance needed: 4950 tokens

// Actual behavior (vulnerable):
// Iteration 1: newAmounts[0] = 1000 - 10 = 990    ✓
// Iteration 2: newAmounts[1] = 1000 - 20 = 980    ✗ (loses extra 10)
// Iteration 3: newAmounts[2] = 1000 - 30 = 970    ✗ (loses extra 20)
// Iteration 4: newAmounts[3] = 1000 - 40 = 960    ✗ (loses extra 30)
// Iteration 5: newAmounts[4] = 1000 - 50 = 950    ✗ (loses extra 40)

// Total user losses: 100 extra tokens
// Contract balance: 4850 (short by 100 tokens)
```

**Fund Depletion Impact:**

```
Contract initially has: 5000 tokens for 5 users
After excessive fees: Users receive only 4850 tokens total
Normally, It should receive 4950 tokens
Protocol debt: 4950 - 4850 = 100 token shortfall
Result: Last users cannot withdraw full amounts
```

**Test Code:**

```solidity
function testCumulativeFeeBug() public {
    uint256[] memory amounts = new uint256[](5);
    amounts[0] = amounts[1] = amounts[2] = amounts[3] = amounts[4] = 1000;

    (uint256 totalFee, uint256[] memory newAmounts) =
        feesManager.calculateFeeAndNewAmountForOneTVS(100, amounts, 5);

    // Bug verification:
    assert(newAmounts[0] == 990); // ✓ Correct
    assert(newAmounts[1] == 980); // ✗ Should be 990
    assert(newAmounts[4] == 950); // ✗ Should be 990
    assert(totalFee == 150);      // ✗ Should be 50
}
```

### Mitigation

Calculate individual fees separately and only deduct the individual fee from each amount.
```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
    public pure returns (uint256 feeAmount, uint256[] memory newAmounts)
{
    newAmounts = new uint256[](length);

    for (uint256 i; i < length;) {
        uint256 individualFee = calculateFeeAmount(feeRate, amounts[i]);
        feeAmount += individualFee;                    // Accumulate for return
        newAmounts[i] = amounts[i] - individualFee;    // Deduct only individual fee
        unchecked { ++i; }
    }
}
```
  