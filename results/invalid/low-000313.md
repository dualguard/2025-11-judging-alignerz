# [000313] Vesting NFT can be burned with zero payout when using wrong `projectId` in `claimTokens()` / `mergeTVS()`
  
  


### Description:
The vesting contract allows a user to burn their vesting NFT and lose access to their allocation if they call [`claimTokens()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L941-L975) or `mergeTVS()` with a wrong but existing `projectId` for a given `nftId`, especially when multiple projects share the same vesting token.

In [`claimTokens()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L941-L975) , the contract derives the `Allocation` solely from the user-supplied `projectId` and `nftId` pair, without verifying that the NFT actually belongs to that project:

```solidity
function claimTokens(uint256 projectId, uint256 nftId) external {
    address nftOwner = nftContract.extOwnerOf(nftId);
    require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());
    bool isBiddingProject = NFTBelongsToBiddingProject[nftId];
    (Allocation storage allocation, IERC20 token) = isBiddingProject 
        ? (biddingProjects[projectId].allocations[nftId], biddingProjects[projectId].token) 
        : (rewardProjects[projectId].allocations[nftId], rewardProjects[projectId].token);

    uint256 nbOfFlows = allocation.vestingPeriods.length;
    uint256 flowsClaimed;
    // loop over flows...
    if (flowsClaimed == nbOfFlows) {
        nftContract.burn(nftId);
        allocation.isClaimed = true;
    }
    token.safeTransfer(msg.sender, claimableAmounts);
}
```

If `projectId` is wrong but points to another project that uses the same `token`, the selected `allocation` will typically be empty (`nbOfFlows == 0`). The loop is skipped, `flowsClaimed == nbOfFlows == 0` holds, so the NFT is burned and `claimableAmounts` remains `0`, resulting in no payout. The real vesting data remains stored under the correct project’s `allocations[nftId]`, but all future operations require ownership of the now-burned NFT, making the user’s claim practically unreachable.

The internal `_merge()` function used by `mergeTVS()` follows the same pattern: it trusts the per-NFT `projectId` passed in `projectIds[]`, looks up `allocations[nftId]` under that project, and unconditionally calls `nftContract.burn(nftId)` even if `TVSToMerge.amounts.length == 0`, which can similarly burn a valid vesting NFT without merging any value.

### Impact:
A mis-parameterized `projectId` can cause a user-owned vesting NFT to be burned with zero tokens paid out, leaving the actual allocated tokens stuck in the contract and effectively unclaimable.

### Recommendation:
Do not rely on user-supplied `projectId` to locate an NFT’s allocation. Instead, maintain an authoritative mapping (e.g. `mapping(uint256 => uint256) projectOfNFT`) set when minting, and in `claimTokens()`, `mergeTVS()`, and `_merge()` either:
- derive the correct `projectId` from `projectOfNFT[nftId]` and ignore any user-supplied value, or  
- enforce `require(projectOfNFT[nftId] == projectId, Invalid_Project_Id());`.


  