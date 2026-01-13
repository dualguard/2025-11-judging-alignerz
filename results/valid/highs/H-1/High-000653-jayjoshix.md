# [000653] Recursive Storage Overwrite in `splitTVS` Causes Permanent Loss of Vested Tokens
  
  ### Summary

In `AlignerzVesting.splitTVS`, using a storage reference to the split NFT's allocation and overwriting it on the first iteration before subsequent iterations read from it will cause a geometric reduction in total allocated tokens for users splitting their TVS NFT, as each percentage after the first is applied to the already-reduced storage value rather than the original post-fee base, resulting in up to 25% permanent loss of claimable tokens for a simple 50/50 split.



### Root Cause

In  https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1067C8-L1101C24 `AlignerzVesting.sol:splitTVS()`, the function obtains a storage pointer `allocation` to `allocations[splitNftId]`, deducts the split fee and writes the post-fee amounts back to `allocation.amounts`, then enters a loop where:

- For iteration `i == 0`, the target `nftId` is the original `splitNftId`, so `newAlloc` points to the same storage slot as `allocation`
- `computeSplitArrays(allocation, percentage, nbOfFlows)` reads the current `allocation.amounts` from storage and computes `alloc.amounts[j] = (allocation.amounts[j] * percentage) / BASISPOINT`
- `assignAllocation(newAlloc, alloc)` overwrites `newAlloc.amounts` (which is `allocation.amounts` for `i == 0`) with this partial share
- For iteration `i > 0`, `computeSplitArrays` again reads from `allocation`, but `allocation.amounts` now contains the reduced value from the first split, not the original post-fee base

```solidity

(Allocation storage allocation, IERC20 token) =
    isBiddingProject
        ? (biddingProjects[projectId].allocations[splitNftId], biddingProjects[projectId].token)
        : (rewardProjects[projectId].allocations[splitNftId], rewardProjects[projectId].token);

uint256[] memory amounts = allocation.amounts;
uint256 nbOfFlows = allocation.amounts.length;

(uint256 feeAmount, uint256[] memory newAmounts) =
    calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);

allocation.amounts = newAmounts;  // post-fee base
token.safeTransfer(treasury, feeAmount);

// ...

for (uint256 i; i < nbOfTVS; ) {
    uint256 percentage = percentages[i];
    sumOfPercentages += percentage;

    uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);
    
    // BUG: reads from current allocation storage, which is mutated on i==0
    Allocation memory alloc = computeSplitArrays(allocation, percentage, nbOfFlows);

    // ...
    Allocation storage newAlloc =
        isBiddingProject
            ? biddingProjects[projectId].allocations[nftId]
            : rewardProjects[projectId].allocations[nftId];

    // Overwrites storage - for i==0, this mutates allocation itself
    assignAllocation(newAlloc, alloc);
    
    // ...
    unchecked { ++i; }
}
```


### Internal Pre-conditions

1. User needs to own a TVS NFT with unclaimed vesting tokens
2. User needs to call `splitTVS()` with percentages array length ≥ 2 (i.e., splitting into at least 2 NFTs)
3. `splitFeeRate` can be any value (including 0); the bug occurs regardless of the fee amount


### External Pre-conditions

none

### Attack Path

1. User owns TVS NFT #100 with 1,000 tokens (post any previous claims) in a single flow
2. User calls `splitTVS(projectId, [5000, 5000], 100)` to split into two equal halves (50% each)
3. Contract deducts split fee (e.g., 0.5% = 5 tokens), leaving 995 tokens in `allocation.amounts[0]`
4. Loop iteration `i = 0`:
   - `nftId = 100` (the original NFT)
   - `computeSplitArrays(allocation, 5000, 1)` computes `995 * 5000 / 10000 = 497.5` (rounded down to 497)
   - `assignAllocation(newAlloc, alloc)` overwrites `allocations[100].amounts[0]` with 497
   - **Critical: `allocation.amounts[0]` is now 497, not 995**
5. Loop iteration `i = 1`:
   - `nftId = 101` (newly minted NFT)
   - `computeSplitArrays(allocation, 5000, 1)` reads the **mutated** `allocation.amounts[0] = 497` and computes `497 * 5000 / 10000 = 248.5` (rounded down to 248)
   - `assignAllocation(newAlloc, alloc)` writes `allocations[101].amounts[0] = 248`
6. Final state:
   - NFT #100: 497 tokens
   - NFT #101: 248 tokens
   - Total: 745 tokens out of the original 995 post-fee
   - **Lost: 250 tokens (25.1% of post-fee TVS) become permanently unclaimable**


### Impact

Users splitting their TVS NFT into multiple parts suffer a guaranteed loss of claimable tokens proportional to the number of splits and the percentages used. For a simple two-way 50/50 split, approximately 25% of the post-fee TVS is lost; for three equal splits (33.33% each), the loss approaches 37%; with more splits or unequal percentages, the geometric reduction compounds further. These tokens become permanently locked in the vesting contract with no NFT entitlement to claim them, effectively burning a portion of the user's vested allocation. No attacker gains these tokens; they are simply destroyed due to the recursive storage mutation.

### PoC

_No response_

### Mitigation

_No response_
  