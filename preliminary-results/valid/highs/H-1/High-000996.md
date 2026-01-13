# [000996] splitTVS reduces the original allocation before finishing all calculations, causing loss of tokens
  
  ### Summary

The splitTVS function is designed to split a single NFT allocation into multiple parts based on percentages. However, it updates the original NFT allocation inside the loop for the first split. This means that in the next iteration, the calculation uses the already reduced allocation, not the original full amount. As a result, some tokens are lost permanently during splitting.

### Root Cause

The issue occurs because the function writes the first split directly to storage (_assignAllocation) during the loop. Storage variables in Solidity are updated immediately, so subsequent iterations read the reduced values rather than the original allocation.

Where it happens in the code (simplified)
```solidity
for (uint256 i; i < nbOfTVS; ++i) {
    uint256 percentage = percentages[i];
    uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);

    Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);

    Allocation storage newAlloc = isBiddingProject
        ? biddingProjects[projectId].allocations[nftId]
        : rewardProjects[projectId].allocations[nftId];

    // Problem: writes to storage immediately
    _assignAllocation(newAlloc, alloc);
}```

allocation is the original NFT stored in storage.

_assignAllocation overwrites it during the first iteration (i == 0).

The next iteration reads the reduced allocation, causing incorrect splits and token loss.

### Internal Pre-conditions

User calls splitTVS with more than one percentage (splitting into multiple NFTs).

### External Pre-conditions

None.

### Attack Path

1. User has 100 tokens.
2. User calls `splitTVS` with percentages [50%, 50%].
3. **Iteration 0**:
   - `percentage` = 50%.
   - `alloc` = 50 tokens.
   - `allocation` is updated to 50 tokens.
4. **Iteration 1**:
   - `percentage` = 50%.
   - `_computeSplitArrays` reads `allocation` (50 tokens).
   - `alloc` = 50% of 50 = 25 tokens.
   - New NFT gets 25 tokens.
5. **Result**:
   - Original NFT: 50 tokens.
   - New NFT: 25 tokens.
   - Total: 75 tokens.
   - **25 tokens are lost forever.**


### Impact

Users lose a significant portion of their funds whenever they split an NFT

### PoC

_No response_

### Mitigation

Read the original allocation amounts into memory *before* the loop and use the memory values for calculations. Do not update the storage `allocation` until *after* all calculations are done
  