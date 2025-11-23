# [000364] H12 Incorrect Split Logic leads to Compounding Reduction
  
  ## Summary

The `AlignerzVesting::splitTVS` function contains an oversight where the storage of the original NFT is updated *during* the loop, affecting subsequent calculations. Specifically, when splitting an NFT, the function iterates through the percentages. For the first iteration (`i=0`), it updates the original NFT's allocation to its new partial amount. For subsequent iterations (`i>0`), it reads the *already reduced* amount from storage to calculate the next share, resulting in significantly lower amounts than intended.

## Root Cause

In `splitTVS`:

```solidity
        for (uint256 i; i < nbOfTVS; ) {
            // ...
            Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
            // ...
            Allocation storage newAlloc = ...; // Points to `allocation` if i=0
            _assignAllocation(newAlloc, alloc); // Updates storage!
```

In `_computeSplitArrays`:

```solidity
    function _computeSplitArrays(Allocation storage allocation, ...) ... {
        uint256[] memory baseAmounts = allocation.amounts; // Reads from storage
        // ...
        alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
```

**The Sequence:**
1.  **Iteration `i=0` (e.g., 50%):**
    *   Reads `baseAmounts` (1000).
    *   Calculates 50% = 500.
    *   Updates `allocation` storage to 500.
2.  **Iteration `i=1` (e.g., 50%):**
    *   Reads `baseAmounts` from `allocation` (which is now 500!).
    *   Calculates 50% of 500 = 250.
    *   Result: NFT 2 gets 250 tokens instead of 500.

## Internal Pre-Conditions

1.  User calls `splitTVS` with multiple percentages.

## External Pre-Conditions

None.

## Impact

**High**. Users lose a significant portion of their funds when splitting NFTs. In a 50/50 split, 25% of the total funds are permanently lost (vanished from the allocation tracking).

## PoC

Trivial.

## Mitigation

Cache the original allocation amounts in memory *before* the loop, and pass this memory array to `_computeSplitArrays` instead of reading from storage inside the loop.



  