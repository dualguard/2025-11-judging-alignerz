# [000944] Wrong Fee Calculation Causes Users to Lose Tokens When Merging NFTs
  
  ### Summary

The `calculateFeeAndNewAmountForOneTVS` function calculates fees incorrectly by using the cumulative total fee instead of the individual fee for each flow. This causes users to lose progressively more tokens on each subsequent flow when merging or splitting NFTs with multiple vesting streams.

### Root Cause

When users merge multiple NFTs, each NFT's vesting allocation becomes a separate flow in the merged NFT. The protocol charges a fee on each flow and should deduct only that flow's fee from its amount. However, the function incorrectly accumulates fees and applies the cumulative total to each flow:
```solidity
 FeesManager.sol

function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate, 
    uint256[] memory amounts, 
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]); //@audit accumulates
        newAmounts[i] = amounts[i] - feeAmount; //@audit subtracts cumulative instead of individual
    }
}
```
The variable feeAmount accumulates across all iterations, but the function subtracts this cumulative value from each flow instead of subtracting only that flow's individual fee.

> [https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169](url)


### Internal Pre-conditions

none

### External Pre-conditions

none

### Attack Path

none

### Impact

Direct financial loss for users:
Every flow after the first loses additional tokens equal to all previous cumulative fees
The more NFTs merged, the worse the loss
Users are effectively charged multiple times for the same fees
Loss is permanent tokens are not recoverable

### PoC

(Walkthrough)
Scenario: Alice merges 2 NFTs with a 1% merge fee:
NFT 1: 5,000 tokens vesting
NFT 2: 1,000 tokens vesting

What should happen:
```text
Flow 0: fee = 5000 × 1% = 50 tokens
        newAmount = 5000 - 50 = 4,950 tokens

Flow 1: fee = 1000 × 1% = 10 tokens
        newAmount = 1000 - 10 = 990 tokens

Total fee to treasury: 60 tokens
Alice receives: 5,940 tokens
```
What actually happens:
```text
Iteration 0:
  individualFee = 50 tokens
  feeAmount = 0 + 50 = 50 (cumulative tracker)
  newAmounts[0] = 5000 - 50 = 4,950 tokens ✓

Iteration 1:
  individualFee = 10 tokens
  feeAmount = 50 + 10 = 60 (cumulative tracker)
  newAmounts[1] = 1000 - 60 = 940 tokens ✗

Total fee to treasury: 60 tokens (correct)
Alice receives: 5,890 tokens (lost 50 tokens)
```
Alice loses 50 additional tokens because the second flow was charged the cumulative fee (60) instead of its own fee (10).
With more flows, the problem compounds:
If Alice merges 3 NFTs with amounts [5000, 1000, 2000]:
```text
Flow 0: 5000 - 50 = 4,950 (correct)
Flow 1: 1000 - 60 = 940 (loses extra 50)
Flow 2: 2000 - 80 = 1,920 (loses extra 60)

Should receive: 7,910 tokens
Actually receives: 7,810 tokens
Loss: 100 tokens
```

### Mitigation

Store and use the individual fee for each flow instead of the cumulative total.
  