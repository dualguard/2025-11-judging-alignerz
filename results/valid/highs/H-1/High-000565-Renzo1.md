# [000565] Storage Reference Mutation in Split Loop Causes Permanent Token Value Loss During TVS Splitting
  
  
## Summary

Mutating the original allocation's storage reference during the first split iteration will cause permanent loss of vested tokens for users splitting their TVS as the same storage reference is reused in subsequent loop iterations, causing each split to use an already-reduced amount as its base, compounding the loss.

&nbsp;

## Root Cause

In [`AlignerzVesting.sol:1083-1105`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1063-L1091), the `splitTVS` function uses a storage reference `allocation` that points to the original `splitNftId` allocation. When `i == 0` in the loop, `newAlloc` (line 1091) points to the same storage location as `allocation` because `nftId == splitNftId` (line 1087). The call to `_assignAllocation(newAlloc, alloc)` at line 1092 overwrites `allocation.amounts` in storage with the first split's reduced amounts. Since `_computeSplitArrays` (line 1114-1142) reads `allocation.amounts` directly from storage at line 1125, all subsequent iterations (`i > 0`) use the already-split amounts as the base, causing each split to be calculated from progressively smaller amounts instead of the original fee-adjusted amount.

```js
    function splitTVS(...) external returns (uint256, uint256[] memory) {
        // ...
        (Allocation storage allocation, IERC20 token) = isBiddingProject ?
        (biddingProjects[projectId].allocations[splitNftId], biddingProjects[projectId].token) :
        (rewardProjects[projectId].allocations[splitNftId], rewardProjects[projectId].token);

        // ...

        uint256 sumOfPercentages;
        for (uint256 i; i < nbOfTVS;) {
            uint256 percentage = percentages[i];
            sumOfPercentages += percentage;

            uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender); // first id is splitNftId
            if (i != 0) newNftIds[i - 1] = nftId; // skip the first id because we want to capture only the new ids
            Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
            NFTBelongsToBiddingProject[nftId] = isBiddingProject ? true : false;
            Allocation storage newAlloc = isBiddingProject ? biddingProjects[projectId].allocations[nftId] : rewardProjects[projectId].allocations[nftId];
            _assignAllocation(newAlloc, alloc);
            // ...
        }
        // ...
    }

```


&nbsp;

## Internal Pre-conditions

1. User needs to own a TVS NFT with `splitNftId` to call `splitTVS`
2. User needs to call `splitTVS` with `percentages` array containing at least 2 elements (to trigger multiple loop iterations)
3. The `splitNftId` must have a non-zero `allocation.amounts` array

&nbsp;

## External Pre-conditions

None required.

&nbsp;

## Attack Path

1. User calls `splitTVS(projectId, percentages, splitNftId)` with multiple percentages (e.g., `[5000, 3000, 2000]` representing 50%, 30%, 20%)
2. Function calculates fee-adjusted amounts and writes them to `allocation.amounts` at line 1071
3. Loop iteration `i == 0`: `_computeSplitArrays` reads from `allocation.amounts` (fee-adjusted original amount) and calculates first split (50% of original)
4. Loop iteration `i == 0`: `_assignAllocation` overwrites the storage `allocation.amounts` with the first split's amounts (50% of original)
5. Loop iteration `i == 1`: `_computeSplitArrays` reads from `allocation.amounts` which now contains the first split's amounts (50% of original) instead of the original amount
6. Loop iteration `i == 1`: Calculates second split as 30% of the already-split amount (50% of original), resulting in only 15% of the original amount instead of 30%
7. Loop iteration `i == 2`: `_computeSplitArrays` reads from `allocation.amounts` which has been further reduced, causing the third split to be calculated from an even smaller base
8. The sum of all split amounts is far less than the original `allocation.amount`, causing permanent token loss

&nbsp;

## Impact

Users suffer permanent loss of a significant portion of their vested tokens when splitting a TVS into multiple parts. The loss compounds with each additional split. For example, splitting a TVS with 1000 tokens into three parts (50%, 30%, 20%) should result in splits of 500, 300, and 200 tokens. Due to the bug, the second and third splits are calculated from the already-reduced first split amount, resulting in total splits of approximately 500, 150, and 100 tokens (750 total instead of 1000), representing a permanent loss of approximately 25% of the user's vested tokens. With more splits, the loss increases further.

&nbsp;

## Proof of Concept
N/A

&nbsp;

## Mitigation

Store the original `allocation.amounts` in memory before the loop and use that memory copy as the base for all split calculations. Alternatively, when `i == 0`, avoid overwriting the original allocation's storage; instead, create a new allocation entry or use a separate memory variable to track the original amounts throughout all iterations. The key fix is to ensure `_computeSplitArrays` always reads from the original fee-adjusted amounts, not from the progressively modified storage reference.


  