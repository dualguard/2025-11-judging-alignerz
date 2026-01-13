# [000126] Incorrect `projectId` handling in `claimTokens` and `mergeTVS` can burn NFTs and lock TVS
  
  ### Summary

`AlignerzVesting` derives vesting allocations solely from `(projectId, nftId)` pairs supplied by the caller, without any on-chain binding between `nftId` and its originating project. Passing an incorrect but existing `projectId` can cause `claimTokens()` and `mergeTVS()` to operate on empty or wrong allocations, burning NFTs and marking allocations as claimed while transferring zero tokens. Severity is medium because users can irrevocably lose NFTs and vesting rights due to a single parameter mistake or malicious front end.

### Root Cause

`claimTokens()` derives the allocation entirely from the caller-provided `projectId`:

```solidity
function claimTokens(uint256 projectId, uint256 nftId) external {
    address nftOwner = nftContract.extOwnerOf(nftId);
    require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());
    bool isBiddingProject = NFTBelongsToBiddingProject[nftId];
    (Allocation storage allocation, IERC20 token) = isBiddingProject
        ? (biddingProjects[projectId].allocations[nftId], biddingProjects[projectId].token)
        : (rewardProjects[projectId].allocations[nftId], rewardProjects[projectId].token);
    uint256 nbOfFlows = allocation.vestingPeriods.length;
    // ...
    if (flowsClaimed == nbOfFlows) {
        nftContract.burn(nftId);
        allocation.isClaimed = true;
    }
    token.safeTransfer(msg.sender, claimableAmounts);
}
```

There is no mapping that ties `nftId` to a unique `projectId`. `NFTBelongsToBiddingProject[nftId]` is a boolean and only distinguishes between the bidding and reward maps, not which project index to use. If the user passes a wrong `projectId` where `allocations[nftId]` is empty:

- `allocation.vestingPeriods.length == 0`, so the for-loop is skipped.
- `flowsClaimed == nbOfFlows == 0`, so the code considers the TVS “fully claimed”.
- `nftContract.burn(nftId)` is called and `allocation.isClaimed` is set.
- `token.safeTransfer(msg.sender, claimableAmounts)` transfers `0` tokens.

The real allocation stored under the correct project’s `allocations[nftId]` remains untouched but becomes unreachable because the NFT is burned and all vesting entry points require `extOwnerOf(nftId)` to succeed.

`mergeTVS()` and `splitTVS()` exhibit the same pattern:

```solidity
function mergeTVS(uint256 projectId, uint256 mergedNftId, uint256[] calldata projectIds, uint256[] calldata nftIds)
    external
    returns (uint256)
{
    address nftOwner = nftContract.extOwnerOf(mergedNftId);
    require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());

    bool isBiddingProject = NFTBelongsToBiddingProject[mergedNftId];
    (Allocation storage mergedTVS, IERC20 token) = isBiddingProject
        ? (biddingProjects[projectId].allocations[mergedNftId], biddingProjects[projectId].token)
        : (rewardProjects[projectId].allocations[mergedNftId], rewardProjects[projectId].token);
    // ...
}
```

In `_merge()`, a similar choice uses `biddingProjects[projectId].allocations[nftId]` or `rewardProjects[projectId].allocations[nftId]` based on `NFTBelongsToBiddingProject[nftId]`. If the caller supplies mismatched `projectId` / `projectIds`, NFTs with real allocations under another project may be merged or burned against empty allocations, effectively destroying the vesting rights.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Suppose NFT `nftId` belongs to reward project `P_real`, with a non-empty `rewardProjects[P_real].allocations[nftId]`.
2. A user calls:
   ```solidity
   claimTokens(P_wrong, nftId);  // P_wrong != P_real, but P_wrong < rewardProjectCount
   ```
3. For `P_wrong`, `rewardProjects[P_wrong].allocations[nftId]` has all arrays length 0.
4. In `claimTokens()`:
   - `nbOfFlows = 0`, `flowsClaimed = 0`, so `flowsClaimed == nbOfFlows` is true.
   - `nftContract.burn(nftId)` is executed, `allocation.isClaimed` is set.
   - `claimableAmounts` is 0, so 0 tokens are transferred.
5. The actual allocation under `P_real` persists in storage but is forever inaccessible since the NFT is burned and all vesting operations require `extOwnerOf(nftId)`.

### Impact

Loss of yield for the user

### PoC

_No response_

### Mitigation


Consider binding each `nftId` to its originating project ID and enforcing that relationship in all TVS operations. 
  