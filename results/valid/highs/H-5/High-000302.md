# [000302] Fee calculation is compounded in Multi-Flow Operations causing excessive token loss
  
  Users with multi-flow NFTs will lose significantly more tokens than the stated fee rate during merge and split operations

# Summary

The incorrect accumulation of fees in [`FeesManager.calculateFeeAndNewAmountForOneTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174) will cause users to lose excessive tokens during merge and split operations as the function subtracts cumulative fees (sum of all previous flows' fees) from each flow instead of only that flow's individual fee, resulting in overcharges that increase quadratically with the number of flows.

# Root Cause

In [`FeesManager.sol`](.[./src/contracts/vesting/feesManager/FeesManager.sol#L167-L177](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169)), the `calculateFeeAndNewAmountForOneTVS()` function accumulates `feeAmount` across loop iterations but incorrectly subtracts this cumulative total from each subsequent flow's amount, rather than subtracting only that flow's individual fee.

```solidity
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length; ) {
        // Accumulates total fee across all iterations
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);

        //@audit-issue Med: Subtracts cumulative feeAmount instead of individual fee
        newAmounts[i] = amounts[i] - feeAmount;
    }
}
```

# Internal Pre-conditions

1. User needs to own an NFT with multiple vesting flows (from previous merges or initial allocation)
2. User needs to call `splitTVS()` or `mergeTVS()` which internally calls `calculateFeeAndNewAmountForOneTVS()`
3. The NFT needs to have at least 2 flows for the bug to manifest (single-flow NFTs are calculated correctly)

# External Pre-conditions
N/A

# Attack Path

1. User owns NFT # 1 with 3 vesting flows: `[10,000, 5,000, 3,000]` tokens
2. User calls `splitTVS()` to split the NFT (1% split fee applies)
3. Function calls `calculateFeeAndNewAmountForOneTVS(feeRate: 100, amounts: [10000, 5000, 3000], length: 3)`
4. Loop iteration 0:
   - Calculates `fee₀ = 10,000 × 1% = 100`
   - Sets `feeAmount = 100` (cumulative)
   - Calculates `newAmounts[0] = 10,000 - 100 = 9,900`
5. Loop iteration 1:
   - Calculates `fee₁ = 5,000 × 1% = 50`
   - Updates `feeAmount = 100 + 50 = 150` (cumulative)
   - Calculates `newAmounts[1] = 5,000 - 150 = 4,850` (overcharged by 100, should've been 4,950)
6. Loop iteration 2:
   - Calculates `fee₂ = 3,000 × 1% = 30`
   - Updates `feeAmount = 150 + 30 = 180` (cumulative)
   - Calculates `newAmounts[2] = 3,000 - 180 = 2,820` (overcharged by 150, should've been 2,970)
7. Returns `newAmounts = [9,900, 4,850, 2,820]` (total: 17,570)
8. User receives split NFTs with these reduced amounts
9. User's child NFTs now have 17,570 tokens instead of expected 17,820 tokens

**Result:** User loses 250 tokens (1.39% effective fee instead of stated 1% fee).

**Worst case with 10 flows of 5,000 tokens each:**

- Expected loss (2% fee): 1,000 tokens
- Actual loss: 5,500 tokens
- Overcharge: 4,500 tokens (9% total loss instead of 2%)

# Impact

Users with multi-flow NFTs suffer excessive token loss during split and merge operations, with the overcharge increasing quadratically as `O(n²)` where n is the number of flows. For a 3-flow NFT with `[10,000, 5,000, 3,000]` tokens and 1% split fee, users lose 250 tokens (1.39% effective rate) instead of the expected 180 tokens (1% rate), representing a 39% overcharge. In worst-case scenarios with 10 flows, users can lose 9% of their tokens instead of the stated 2% fee, a 350% overcharge.


# Mitigation

Fix the fee calculation to deduct only each flow's individual fee instead of the cumulative total:

```solidity
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    newAmounts = new uint256[](length);  // Initialize array explicitly

    for (uint256 i; i < length; ) {
        uint256 fee = calculateFeeAmount(feeRate, amounts[i]);

        // Accumulate total fee for protocol revenue tracking
        feeAmount += fee;

        newAmounts[i] = amounts[i] - fee;

        unchecked {
            ++i;
        }
    }
}
```
  