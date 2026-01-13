# [000233] `splitTVS` function is dysfunctional due to out-of-bounds write in `_computeSplitArrays`
  
  ### Summary

The `splitTVS` function, which is designed to allow users to divide their vesting NFTs into multiple smaller NFTs, is non-functional. The function relies on an internal helper, `_computeSplitArrays`, to calculate the new allocations for the split NFTs. This helper function attempts to write to uninitialized dynamic arrays in memory, causing an out-of-bounds access error. This error guarantees that any call to `splitTVS` will revert, rendering a core feature of the contract completely unusable and causing users to waste gas on failed transactions.

### Root Cause

The root cause of this critical bug lies in the implementation of the `_computeSplitArrays` function. In `AlignerzVesting.sol: 1113 - 1141`

https://github.com/dualguard/2025-11-alignerz-Ayomiposi233/blob/7b05b7b1bbb71e3e6957270e83365a936945ea5d/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113-L1141

1. The function declares a return variable `Allocation memory alloc`. When a struct containing dynamic arrays is created in `memory`, its arrays (`amounts`, `vestingPeriods`, etc.) are initialized with a length of `0`.
2. The function then enters a `for` loop that iterates from `j = 0` to `nbOfFlows - 1`.
3. Inside the loop, it attempts to populate the arrays of the `alloc` struct.
4. On the very first iteration (`j=0`), this line tries to write to index `0` of the `alloc.amounts` array. Since this array has a length of `0`, this is an out-of-bounds memory access. Solidity prevents such operations by reverting the transaction.

### Internal Pre-conditions

- A user owns a vesting NFT from either a `BiddingProject` or a `RewardProject`.
- The vesting NFT has at least one vesting flow (`allocation.amounts.length > 0`).

### External Pre-conditions

- The owner of a valid vesting NFT calls the `splitTVS` function with valid parameters.

### Attack Path

1. __Setup__: A user owns a vesting NFT, `splitNftId`, which represents a single vesting flow.
2. __Action__: The user calls `splitTVS(projectId, [5000, 5000], splitNftId)`, intending to split their NFT into two equal 50% shares.
3. __Execution__:
- The `splitTVS` function begins execution and eventually calls the internal function `_computeSplitArrays`.
- Inside `_computeSplitArrays`, the `Allocation memory alloc` struct is created. At this point, `alloc.amounts.length` is `0`.
- The `for` loop starts with `j = 0`.
- The code attempts to execute `alloc.amounts[0] = (baseAmounts[0] * 5000) / BASIS_POINT;`.
4. __Outcome__: The transaction reverts with an out-of-bounds error because it's trying to write to index `0` of a zero-length memory array. This will happen for any valid input to `splitTVS`.

### Impact

This bug renders the entire `splitTVS` feature completely non-functional. It is a high-severity issue because it breaks a core advertised functionality of the contract. Users attempting to use this feature will have their transactions reverted, leading to wasted gas fees, without any way to successfully split their NFTs.

### PoC

_No response_

### Mitigation

Properly initialize the dynamic arrays in the `alloc` memory struct by allocating them with the correct length (`nbOfFlows`) before the loop begins. This ensures that the memory is available to be written to.
  