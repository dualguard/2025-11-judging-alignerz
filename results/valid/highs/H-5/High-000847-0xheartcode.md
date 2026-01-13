# [000847] Critical Fee Calculation Bug Causes Incorrect Fee Deduction
  
  ## Summary

The `calculateFeeAndNewAmountForOneTVS` function in `FeesManager.sol` contains a critical logic error that causes exponentially increasing fee deductions across multiple token flows in a TVS, leading to significant user fund loss.

## Vulnerability Detail

The vulnerability exists in the fee calculation logic for TVS operations (splitting/merging):

**Location**: [`FeesManager.sol:169-174`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174)

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount; // 🚨 BUG: Should be amounts[i] - currentFee
    }
}
```

**Root Cause**: The function deducts the cumulative `feeAmount` from each flow instead of deducting only the individual flow's fee.

**Correct Logic Should Be**:
```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        uint256 currentFee = calculateFeeAmount(feeRate, amounts[i]);
        feeAmount += currentFee;
        newAmounts[i] = amounts[i] - currentFee; // Correct: Deduct only this flow's fee
    }
}
```

### Attack Mechanism

1. **TVS Creation**: User has a TVS with multiple token flows (e.g., 3 flows of 1000 tokens each)
2. **Split/Merge Operation**: User calls `splitTVS` or `mergeTVS` triggering fee calculation
3. **Exponential Deduction**: 
   - Flow 0: 1000 - 50 = 950 tokens (5% fee = 50)
   - Flow 1: 1000 - 100 = 900 tokens (cumulative fee = 100)  
   - Flow 2: 1000 - 150 = 850 tokens (cumulative fee = 150)
4. **Fund Loss**: User loses 300 tokens instead of expected 150 tokens

## Impact

**High Severity**: Direct fund loss without external conditions

- **Exponential Fund Loss**: Users lose far more tokens than the intended fee rate
- **Multi-Flow Impact**: The more flows in a TVS, the greater the loss
- **Systematic Exploitation**: Affects all TVS split/merge operations
- **User Experience**: Could make TVS operations economically unviable

**Example Scenario**:
- 10-flow TVS with 1000 tokens each, 5% fee rate
- Expected total fee: 500 tokens (5% of 10,000)  
- Actual total deducted: 2,750 tokens (27.5% effective rate)
- **User loses 2,250 additional tokens**

## Code Snippet

**Vulnerable implementation** - [`FeesManager.sol:169-174`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174):
```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount; // BUG: Cumulative deduction
        unchecked {
            ++i;
        }
    }
}
```

**Usage in AlignerzVesting.sol**:
```solidity
// Line 1013: Split operation
(uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
mergedTVS.amounts = newAmounts;

// Line 1069: Merge operation  
(uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);
allocation.amounts = newAmounts;
```

**Code References**:
- [`AlignerzVesting.sol:1013`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1013) - mergeTVS operation
- [`AlignerzVesting.sol:1069`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1069) - splitTVS operation

## Recommendation

Fix the fee calculation logic to deduct individual fees per flow:

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    newAmounts = new uint256[](length); // Initialize array
    for (uint256 i; i < length;) {
        uint256 currentFee = calculateFeeAmount(feeRate, amounts[i]);
        feeAmount += currentFee;
        newAmounts[i] = amounts[i] - currentFee; // Fix: Deduct only this flow's fee
        unchecked {
            ++i;
        }
    }
}
```
  