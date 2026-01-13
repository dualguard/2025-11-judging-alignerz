# [000773] Not updating `allocationOf[nftId]` causes inaccurate dividend calculations for merged TVSs
  
  ### Summary

Not updating `allocationOf[nftId]` after merges will cause inaccurate dividend calculations for merged TVSs as the dividend distributor reads stale pre-merge allocations while the real vesting state lives only in the per-project `allocations` mappings.

### Root Cause

In `AlignerzVesting:L1001-L1023` the `mergeTVS` function updates only the project-specific `allocations` storage (`biddingProjects[projectId].allocations[mergedNftId]` or `rewardProjects[projectId].allocations[mergedNftId]`) and never syncs the public `allocationOf` mapping for `mergedNftId`.

```solidity 
File: AlignerzVesting.sol

function mergeTVS(uint256 projectId, uint256 mergedNftId, uint256[] calldata projectIds, uint256[] calldata nftIds) external returns(uint256) {
    address nftOwner = nftContract.extOwnerOf(mergedNftId);
    require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());
    
    bool isBiddingProject = NFTBelongsToBiddingProject[mergedNftId];
    (Allocation storage mergedTVS, IERC20 token) = isBiddingProject ?
        (biddingProjects[projectId].allocations[mergedNftId], biddingProjects[projectId].token) :
        (rewardProjects[projectId].allocations[mergedNftId], rewardProjects[projectId].token);

    uint256[] memory amounts = mergedTVS.amounts;
    uint256 nbOfFlows = mergedTVS.amounts.length;
    (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
>>  mergedTVS.amounts = newAmounts;

    uint256 nbOfNFTs = nftIds.length;
    require(nbOfNFTs > 0, Not_Enough_TVS_To_Merge());
    require(nbOfNFTs == projectIds.length, Array_Lengths_Must_Match());

    for (uint256 i; i < nbOfNFTs; i++) {
        feeAmount += _merge(mergedTVS, projectIds[i], nftIds[i], token);
    }
    token.safeTransfer(treasury, feeAmount);
    emit TVSsMerged(projectId, isBiddingProject, nftIds, mergedNftId, mergedTVS.amounts, mergedTVS.vestingPeriods, mergedTVS.vestingStartTimes, mergedTVS.claimedSeconds, mergedTVS.claimedFlows);
    return mergedNftId;
}
```

By contrast, on mint / split paths the contract explicitly writes to `allocationOf`:

```solidity
File: AlignerzVesting.sol

function _claimRewardTVS(uint256 rewardProjectId, address kol) internal {
    ...
    uint256 nftId = nftContract.mint(kol);
    rewardProject.allocations[nftId].amounts.push(amount);
    ...
>>  allocationOf[nftId] = rewardProject.allocations[nftId];
    ...
}


function claimNFT(uint256 projectId, uint256 poolId, uint256 amount, bytes32[] calldata merkleProof)
    external
    returns (uint256)
{
    ...
    uint256 nftId = nftContract.mint(msg.sender);
    biddingProject.allocations[nftId].amounts.push(amount);
    ...
    biddingProject.allocations[nftId].token = biddingProject.token;
    NFTBelongsToBiddingProject[nftId] = true;
>>  allocationOf[nftId] = biddingProject.allocations[nftId];
    ...
}


function splitTVS(...) external returns (uint256, uint256[] memory) {
    ...
    uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);
    ...
    Allocation storage newAlloc = isBiddingProject ? biddingProjects[projectId].allocations[nftId] : rewardProjects[projectId].allocations[nftId];
    _assignAllocation(newAlloc, alloc);
>>  allocationOf[nftId] = newAlloc;
    ...
}
```

The dividend distributor relies solely on `allocationOf` to compute unclaimed amounts:

```solidity
File: A26ZDividendDistributor.sol

function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
    uint256 len = nft.getTotalMinted();
    for (uint i; i < len;) {
        (, bool isOwned) = safeOwnerOf(i);
        if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
        unchecked {
            ++i;
        }
    }
}

function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
>>  IAlignerzVesting.Allocation memory alloc = vesting.allocationOf(nftId);

    if (address(token) != address(alloc.token)) {
        return 0;
    }

    uint256 len = alloc.amounts.length;
    for (uint256 i; i < len; ) {
        if (alloc.claimedFlows[i]) {
            unchecked { ++i; }
            continue;
        }

        if (alloc.claimedSeconds[i] == 0) {
            amount += alloc.amounts[i];
        } else {
            uint256 claimedAmount = alloc.claimedSeconds[i] * alloc.amounts[i] / alloc.vestingPeriods[i];
            uint256 unclaimedAmount = alloc.amounts[i] - claimedAmount;
            amount += unclaimedAmount;
        }

        unchecked { ++i; }
    }

    unclaimedAmountsIn[nftId] = amount;
}
```

Solidity storage-to-storage assignment for structs copies array contents, not references, so after the initial `allocationOf[nftId] = ...` the per-project `allocations[...]` and `allocationOf[nftId]` are independent; later writes to `mergedTVS` in `mergeTVS` / `_merge` do not automatically propagate into `allocationOf`.


### Internal Pre-conditions

1. At least one TVS NFT exists and has its `allocationOf[nftId]` correctly populated through `claimRewardTVS`, `claimNFT` or `splitTVS`.  
2. A user performs a merge by calling `mergeTVS` on `mergedNftId` with one or more `nftIds` to merge, so that the real vesting state of `mergedNftId` in `biddingProjects[projectId].allocations[mergedNftId]` or `rewardProjects[projectId].allocations[mergedNftId]` changes.

### External Pre-conditions

1. The owner calls `setAmounts()` / `setDividends()` or `setUpTheDividends()` after merges have taken place, so that dividend calculations run over `allocationOf` instead of the updated per-project `allocations`.

### Attack Path

1. Admin deploys vesting and dividend distributor, and sets up normal vesting flows so that users hold TVS NFTs with allocations; at this point `allocationOf[nftId]` and the per-project `allocations[...]` are in sync.  
2. A user holds multiple TVS NFTs and calls `mergeTVS` to merge them into a single `mergedNftId`; internally `mergedTVS` (the per-project `allocations[mergedNftId]`) is updated and source NFTs are burned, but `allocationOf[mergedNftId]` is not updated.  
3. Later, admin funds the dividend distributor with stablecoin and calls `setAmounts()` and `setDividends()`; inside those functions, `getTotalUnclaimedAmounts` and `getUnclaimedAmounts` only read `vesting.allocationOf(nftId)` for each live NFT ID.  
4. The merged flows that were pushed into `mergedTVS` inside `mergeTVS` / `_merge` are invisible to `allocationOf[mergedNftId]`, and burned NFT IDs are skipped by `safeOwnerOf`, so those merged-units of TVS are not counted in `totalUnclaimedAmounts` and never get a proportional share of dividends.  
5. As a result, the merged TVS holder receives dividends as if only the original (pre-merge) allocation existed for `mergedNftId`, while other TVS holders see a slightly higher dividend per visible unit and the stablecoin corresponding to the missing share remains in the distributor contract (and can be later withdrawn as “stuck tokens”).


### Impact

The holders of merged TVSs receive systematically lower dividends than they should for the flows merged into their `mergedNftId`, because those merged flows are never reflected in `allocationOf` and therefore never participate in dividend calculations, leading to an economic loss for those holders without a corresponding explicit fee or configuration.

### PoC

I tried but it's difficult without touching the original code as there is another bug related to `allocationOf` getter which is reported in another submission.

### Mitigation

After `mergeTVS` finishes updating `mergedTVS` (the per‑project `allocations[mergedNftId]`), the contract should also refresh the public snapshot by assigning `allocationOf[mergedNftId] = mergedTVS`.  

  