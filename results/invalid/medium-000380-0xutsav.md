# [000380] Unfair Dividend Distribution Due to Incorrect claimed Amount Calculation in A26ZDividendDistributor:getUnclaimedAmounts()
  
  ## Summary

The A26ZDividendDistributor contract incorrectly calculates unclaimed amounts for split NFTs, creating systematic unfairness in dividend distributions. Users who split their TVS positions after claiming tokens gain inflated dividend eligibility at the expense of other legitimate dividend recipients.

---

## Root Cause

**Primary Issue**: The `getUnclaimedAmounts()` function in A26ZDividendDistributor uses reduced `amounts[i]` values from split operations to calculate claimed portions, leading to inflated unclaimed amount calculations.

**Vulnerable Code Location:**
```solidity
// A26ZDividendDistributor.sol:getUnclaimedAmounts()
uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
uint256 unclaimedAmount = amounts[i] - claimedAmount;
```

After Splitting/Merging TVS, the amount is updated to (amount-fee) which is used there in `A26ZDividendDistributor:getUnclaimedAmounts()` to get claimed amount 
[getUnclaimedAmounts](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L153)

1. **Initial State**: User has 1000 token TVS with 10-day vesting period
2. **Day 5**: User claims 500 tokens (50% vested)
3. **Day 5**: User splits TVS into 2 NFTs(50-50) with 1% split fee
4. **Split Result**: New amount set to 990 tokens, each NFT gets amounts[i] = 495 tokens
5. **Dividend Calculation Per NFT**: (5 days × 495 tokens) ÷ 10 days = 247.5 tokens "claimed"
6. **Unclaimed Per NFT**: 495 - 247.5 = 247.5 tokens
7. **Total Reported Unclaimed**: 247.5 × 2 = 495 tokens
8. **Actual Unclaimed**: 490 tokens (500 original unclaimed - 10 fee)
9. **False Dividend Advantage**: +5 tokens of inflated unclaimed eligibility

---

## Impact
- Users who split gain advantage in dividend calculations  
- Non-split users receive reduced dividend shares
- Total dividend pool gets redistributed unfairly

---

## Attack Path

### Exploitation Steps:
1. User obtains TVS allocation with vesting schedule
2. User partially claims tokens during vesting period
3. User splits TVS, and new reduced amount is set to (amount-fee)
4. **Result**: Dividend distributor miscalculates unclaimed amounts
   - Uses reduced `amounts[i]` from split operation
   - Applies same `claimedSeconds[i]` to calculate "claimed" portion
   - Creates inflated unclaimed amount calculations
5. User gains unfair dividend share at expense of other users

## Mitigation
Use different variable to track claimed amount 
  