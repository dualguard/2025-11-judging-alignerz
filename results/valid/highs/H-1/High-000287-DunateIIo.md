# [000287] Incorrect Storage Overwrite in splitTVS() Leading To User's TVS to Vanish in some cases and Guaranteed Loss on Every Execution
  
  ### Summary

_(This report is AI written, with a full review by me to finish by deadline)_
The splitTVS function  contains a critical vulnerability where it overwrites its source storage array during the first iteration of its loop. When a user splits an NFT, the function correctly calculates the token amounts for the first new allocation (which reuses the original splitNftId ). However, it immediately writes this new split amount back into the original NFT's storage slot. This corrupted, smaller amount is then incorrectly used as the base for calculating all subsequent splits in the same transaction, leading to an incorrect and progressively diminishing distribution of tokens.

### Root Cause

1. Storage Pointer Initialization: Before the loop, the variable allocation is declared as a storage pointer to the original NFT's data.

https://github.com/dualguard/2025-11-alignerz-DunateIIo/blob/fe542df71d42e3a855f2b014032440ccc2b40da4/protocol/src/contracts/vesting/AlignerzVesting.sol#L1063-L1065



2. First Loop Iteration (i == 0):
- The nftId is intentionally set to be the original splitNftId.
- A new storage pointer, `newAlloc`, is created. Because `nftId == splitNftId`, `newAlloc` points to the exact same storage slot as `allocation`.
- `_computeSplitArrays` is called. It correctly reads the post-fee amounts from `allocation` and calculates the new (smaller) split amounts, returning them in a memory variable `alloc`.
- `_assignAllocation` is called. It takes the split amounts from the memory variable `alloc` and writes them into storage using the `newAlloc` pointer.
https://github.com/dualguard/2025-11-alignerz-DunateIIo/blob/fe542df71d42e3a855f2b014032440ccc2b40da4/protocol/src/contracts/vesting/AlignerzVesting.sol#L1082-L1091

3. The Overwrite: This write operation  overwrites the data in `allocation.amounts` with the new, smaller split amount, as both `allocation` and `newAlloc` point to the same storage location.
4. Subsequent Loop Iterations (`i > 0`):
- A new `nftId` is minted , and `newAlloc` points to a new, empty storage slot.
- `_computeSplitArrays` is called again. However, when it reads from the `allocation` storage pointer, it is no longer reading the original post-fee amounts. It is reading the corrupted, smaller amounts that were written during the first loop.
- This corrupted data is then used as the basis for all subsequent split calculations, leading to exponentially decreasing token amounts for each new NFT.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Initial State: The `allocation` storage pointer points to `splitNftId`'s data: `amounts = [1,000,000]`.
2. Fee Calculation: The 1% fee is calculated, and the `allocation.amounts` array in storage is updated.
  - `allocation.amounts` (in storage) is now: `[990,000]`
3. Loop 1 (i=0): 1% Split

- The `percentage` is 100 (1%).
- The `nftId` is set to the original `splitNftId`.
- `_computeSplitArrays` reads the `allocation.amounts` (990,000) and calculates 1% of it.
- Calculated Amount: 990,000 * 1% = 9,900 tokens.
- `_assignAllocation` is called. It writes this new `[9,900]` array back into storage for `splitNftId`.
- Storage Overwrite: The `allocation` pointer (which still points to `splitNftId`) now reads `amounts = [9,900]`.

4. Loop 2 (i=1): 99% Split 

- The `percentage` is 9900 (99%).
- A new NFT (`newNftId_1`) is minted.
- `_computeSplitArrays` is called. It reads the `allocation.amounts` to use as its base.
- Crucial Error: It reads the overwritten value of 9,900 (not 990,000).
- Calculated Amount: 9,900 * 99% = 9,801 tokens.
- `_assignAllocation` writes `[9,801] `into the storage for `newNftId_1`.

### Impact

Incorrect Token Distribution: All newly minted NFTs (from the second split onwards) will be assigned an incorrect and significantly lower amount of tokens than intended. For example, a 1% - 99% split would result in the first NFT getting 1%, and the second NFT getting also about 1%.

Token Loss: The total amount of tokens allocated across the new set of split NFTs will not equal the original NFT's total. The "missing" tokens are not stolen but become permanently locked in the contract, as they are no longer associated with any vesting allocation.



### PoC

_No response_

### Mitigation

_No response_
  