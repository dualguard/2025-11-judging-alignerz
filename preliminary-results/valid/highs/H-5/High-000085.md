# [000085] Cumulative fee subtraction causes incorrect allocation amounts
  
  ### Summary

The cumulative fee subtraction in `calculateFeeAndNewAmountForOneTVS ` will cause users to receive drastically reduced allocations when merging or splitting NFTs, as the function incorrectly subtracts the accumulated total fees from each individual flow instead of subtracting only that flow's fee.

### Root Cause

In [FeesManager.sol:171-172](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L171-L172), the `calculateFeeAndNewAmountForOneTVS()` accumulates fees in `feeAmount` across all iterations, but then subtracts this cumulative total from each individual flow amount. This means that: 

Flow 0:` newAmounts[0] = amounts[0] - fee[0]` (correct)
Flow 1: `newAmounts[1] = amounts[1] - (fee[0] + fee[1])` (wrong - subtracts too much)
Flow 2: `newAmounts[2] = amounts[2] - (fee[0] + fee[1] + fee[2])` (wrong - subtracts even more)
Each subsequent flow has increasingly excessive fees deducted.

### Internal Pre-conditions

1. User needs to call `mergeTVS()` or `splitTVS()` with allocations containing multiple flows
2. The allocation must have `length >= 2` for the bug to manifest

### External Pre-conditions

None

### Attack Path

1. User has an NFT with 3 flows of 1000 tokens each
2. User calls `mergeTVS()` to merge this NFT (assume 1% fee = 10 tokens per flow)
3. Function calls `calculateFeeAndNewAmountForOneTVS()` 
4. Loop calculates:

    - Flow 0:` newAmounts[0] = 1000 - 10 = 990` correct
    - Flow 1: `newAmounts[1] = 1000 - (10 + 10) = 980` incorrect (should be 990)
    - Flow 2: `newAmounts[2] = 1000 - (10 + 10 + 10) = 970` incorrect (should be 990)

5. User receives 990 + 980 + 970 = 2940 tokens instead of 2970 tokens
6. User loses 30 tokens (1% of total) due to incorrect fee calculation

### Impact

Users with multi-flow allocations lose more tokens than intended, with the exact loss depending on the distribution of amounts across flows. The cumulative subtraction causes each flow to lose not just its own fee, but also the accumulated fees from all previous flows.

### PoC

The logic is evident from code inspection.

### Mitigation

Calculate and substract each flow's fee independently:

```diff
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    newAmounts = new uint256[](length);
    for (uint256 i; i < length; ) {
        uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
+        feeAmount += fee;                  // accumulate total fee
+        newAmounts[i] = amounts[i] - fee;  // subtract fee per flow
        unchecked { ++i; }
    }
}
```
  