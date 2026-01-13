# [000685] Incorrect use of storage pointer in `splitTVS` loop results in cascading token loss for users
  
  ### Summary

The `splitTVS` function incorrectly uses a storage pointer (`allocation`) as the source of truth for split calculations while simultaneously modifying the underlying data pointed to by that variable. Specifically, in the first iteration of the loop (`i=0`), the function overwrites the original NFT's allocation amounts. In subsequent iterations (`i>0`), the function reads from this *already-reduced* amount to calculate the shares for new NFTs, rather than reading from the original total. This leads to a cascading reduction in distributed tokens, causing users to permanently lose a significant portion of their funds.

### Root Cause

In `AlignerzVesting.sol`, the `splitTVS` function declares a storage pointer `allocation` referencing the original NFT's data:

```solidity
Allocation storage allocation = ...; // Points to storage of splitNftId
```

Inside the loop:
1.  `_computeSplitArrays` is called passing `allocation` (the pointer) as the source.
2.  When `i == 0`, the function reuses `splitNftId`. The destination `newAlloc` points to the **same storage location** as `allocation`.
3.  `_assignAllocation(newAlloc, alloc)` overwrites the storage data with the reduced split amount (e.g., 50% of original).
4.  When `i > 0`, `_computeSplitArrays` is called again with the same `allocation` pointer. It now reads the **new, reduced value** (50%) instead of the original value (100%) to calculate the next share.

### Internal Pre-conditions

1.  A user calls `splitTVS` to split a TVS into two or more parts (e.g., 50% and 50%).

### External Pre-conditions

_No response_

### Attack Path

1.  Assume a user has a TVS with `1000` tokens and wants to split it 50/50.
2.  The user calls `splitTVS` with `percentages = [5000, 5000]` (basis points).
3.  **Iteration 1 (i=0):**
    *   Source: `allocation.amounts` reads `1000`.
    *   Calc: `1000 * 50% = 500`.
    *   Write: The original TVS amounts are updated to `500`.
4.  **Iteration 2 (i=1):**
    *   Source: `allocation.amounts` (pointer) now reads `500` (Modified in Step 3).
    *   Calc: `500 * 50% = 250`. (Should have been `1000 * 50% = 500`).
    *   Write: The new NFT is minted with `250` tokens.
5.  **Result:** User ends up with `500 + 250 = 750` tokens. **250 tokens are permanently lost.**

### Impact

Users suffer a guaranteed and irreversible loss of funds whenever they use the split functionality. The loss magnitude increases with the number of splits (e.g., splitting into 4 parts of 25% results in receiving only ~36% of the original funds instead of 100%).

### PoC

_No response_

### Mitigation

Create a memory copy (snapshot) of the original allocation data before entering the loop, and use this read-only memory copy as the source for all calculations.
  