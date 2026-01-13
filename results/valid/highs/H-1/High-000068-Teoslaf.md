# [000068] Storage Corruption in splitTVS() Causes Permanent Fund Loss
  
  ## Summary

The [`splitTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054) function contains a storage corruption bug that causes users to lose portion of their tokens. The bug occurs because the function modifies storage during the first iteration of the split loop, then reads from that modified storage in subsequent iterations, causing incorrect percentage calculations. The severity of the loss depends on the split percentages chosen - smaller first percentages result in catastrophic losses.

### Root Cause

The `splitTVS()` function:

1. Deducts the 0.5% fee and updates storage: `allocation.amounts = newAmounts`
2. In the first iteration (i=0), reuses the original NFT ID
3. Calculates the first percentage and overwrites the same storage
4. In subsequent iterations, reads from the already modified storage
5. Calculates remaining percentages based on the modified values instead of the original

### Code Flow

```solidity
function splitTVS(uint256 projectId, uint256[] calldata percentages, uint256 splitNftId)
    external returns (uint256, uint256[] memory) {

    // Get storage pointer to original NFT
    Allocation storage allocation = allocations[splitNftId];

    // Step 1: Deduct fee and update storage
    uint256[] memory amounts = allocation.amounts;  // [1000, 2000, 3000]
    (feeAmount, newAmounts) = calculateFee(amounts);  // [995, 1990, 2985]
    allocation.amounts = newAmounts;  // Storage now: [995, 1990, 2985]

    // Step 2: Loop through percentages
    for (uint256 i = 0; i < nbOfTVS; i++) {
        // i=0: Reuse original NFT
        uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);

        // Reads from 'allocation' storage
        Allocation memory alloc = _computeSplitArrays(allocation, percentages[i], nbOfFlows);

        // For i=0, overwrites same storage as 'allocation'
        Allocation storage newAlloc = allocations[nftId];
        _assignAllocation(newAlloc, alloc);

        // After i=0: allocation.amounts modified to first percentage
        // After i=1+: Calculates from modified storage instead of original
    }
}
```

## Impact

Expected Distribution (after 0.5% fee):

```bash
Total: 5970 tokens

nft 123 (20%): 1194 tokens
nft 456 (30%): 1791 tokens
nft 789 (50%): 2985 tokens
```

Actual Distribution:

```bash
Total: 2147 tokens

nft 123 (20%): 1194 tokens (correct)
nft 456 (6%):   357 tokens (1434 tokens lost)
nft 789 (10%):  596 tokens (2389 tokens lost)

Missing: 3823 tokens (64.03% of total)
```

## Proof of Concept

You can add this test in `protocol/test/SplitTVSStorageCorruption.t.sol`.

Note: I created a mock from the original cotract beacuse we hat to fix few bugs to demonstrate this one

See separate findings:

1. Uninitialized Array in calculateFeeAndNewAmountForOneTVS() Causes Complete DoS
2. Infinite Loop DoS
3. Incorrect fee logic overcharges users

### Running the Test

```bash
forge test --match-test test_SplitTVS_StorageCorruption -vv
```

### Test Output

```
nft 123 (20%): got 1194 expected 1194
nft 456 (30%): got 357 expected 1791
nft 789 (50%): got 596 expected 2985
total loss: 3823

[FAIL: expected failure: users lose funds due to storage corruption bug: 2147 != 5970]
```

## Recommended Mitigation

Create a complete memory copy of the allocation (including all fee-deducted amounts) before the split loop begins. Calculate all split percentages from this memory copy instead of reading from storage. Modify `_computeSplitArrays()` to accept a memory allocation parameter instead of a storage pointer.

  