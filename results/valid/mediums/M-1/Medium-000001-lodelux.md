# [000001] Merging TVSs keeps allocationOf stale and underpays dividends for merged TVS NFT holders
  
  ### Summary

The failure to keep the global `allocationOf` mapping in sync with per-project allocations during [`AlignerzVesting::mergeTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002-L1026) will cause merged TVS NFT holders to be under-represented in `A26ZDividendDistributor` as any flows merged into an existing NFT are not reflected in `allocationOf`, so downstream dividend accounting that relies on this mapping will underestimate the unclaimed amounts for merged NFTs and misallocate dividends in favor of unmerged TVSs.

### Root Cause

The vesting contract maintains two parallel sources of truth for each TVS allocation: per-project `allocations` mappings (inside bidding/reward projects) and a global `allocationOf` mapping that external consumers (such as the dividend distributor) read from. When merging TVSs, only the per-project `allocations` entry is updated; the global `allocationOf` entry for the surviving NFT remains at its pre-merge value:

```solidity
contract AlignerzVesting {
    struct Allocation {
        uint256[] amounts;
        uint256[] vestingPeriods;
        uint256[] vestingStartTimes;
        uint256[] claimedSeconds;
        bool[] claimedFlows;
        bool isClaimed;
        IERC20 token;
        uint256 assignedPoolId;
    }

@>  mapping(uint256 => Allocation) public allocationOf;
    // ...
}
```

Merges are performed by `AlignerzVesting::mergeTVS`, which mutates the storage allocation referenced from the project-specific `allocations` mapping but never writes the result back to `allocationOf`:

```solidity
function mergeTVS(
    uint256 projectId,
    uint256 mergedNftId,
    uint256[] calldata projectIds,
    uint256[] calldata nftIds
) external returns (uint256) {

    // ...

    bool isBiddingProject = NFTBelongsToBiddingProject[mergedNftId];
    (Allocation storage mergedTVS, IERC20 token) = isBiddingProject
        ? (
            biddingProjects[projectId].allocations[mergedNftId],
            biddingProjects[projectId].token
        )
        : (
            rewardProjects[projectId].allocations[mergedNftId],
            rewardProjects[projectId].token
        );

    //...

    for (uint256 i; i < nbOfNFTs; i++) {
        feeAmount += _merge(mergedTVS, projectIds[i], nftIds[i], token);
    }

    // ...
}
```

Within `_merge`, additional flows from each NFT being merged are appended to the same `mergedTVS` storage struct, and the source NFTs are burned, but again `allocationOf` is never updated for either the surviving or burned IDs:

```solidity
function _merge(
    Allocation storage mergedTVS,
    uint256 projectId,
    uint256 nftId,
    IERC20 token
) internal returns (uint256 feeAmount) {
    require(
        msg.sender == nftContract.extOwnerOf(nftId),
        Caller_Should_Own_The_NFT()
    );

    bool isBiddingProjectTVSToMerge = NFTBelongsToBiddingProject[nftId];
    (Allocation storage TVSToMerge, IERC20 tokenToMerge) =
        isBiddingProjectTVSToMerge
            ? (
                biddingProjects[projectId].allocations[nftId],
                biddingProjects[projectId].token
            )
            : (
                rewardProjects[projectId].allocations[nftId],
                rewardProjects[projectId].token
            );
    require(address(token) == address(tokenToMerge), Different_Tokens());

    uint256 nbOfFlowsTVSToMerge = TVSToMerge.amounts.length;
    for (uint256 j = 0; j < nbOfFlowsTVSToMerge; j++) {
        uint256 fee = calculateFeeAmount(mergeFeeRate, TVSToMerge.amounts[j]);
        mergedTVS.amounts.push(TVSToMerge.amounts[j] - fee);
        mergedTVS.vestingPeriods.push(TVSToMerge.vestingPeriods[j]);
        mergedTVS.vestingStartTimes.push(TVSToMerge.vestingStartTimes[j]);
        mergedTVS.claimedSeconds.push(TVSToMerge.claimedSeconds[j]);
        mergedTVS.claimedFlows.push(TVSToMerge.claimedFlows[j]);
        feeAmount += fee;
    }
@>  nftContract.burn(nftId);
}
```

As a result:

- The per-project `allocations[mergedNftId]` struct correctly reflects all merged flows (minus fees), but `allocationOf[mergedNftId]` still encodes only the original, pre-merge flows.
- For each burned `nftId`, `allocationOf[nftId]` is left pointing to the old allocation even though the NFT no longer exists.

the dividend distributor and any other consumer that relies on `allocationOf[nftId]` as the source of truth for a TVS will read stale data after merges. In particular, `A26ZDividendDistributor::getUnclaimedAmounts` uses `vesting.allocationOf(nftId).amounts`  to compute per-NFT unclaimed amounts; merged NFTs therefore appear to have only their original flows, while flows merged in from now-burned NFTs are invisible, leading to under-allocated dividends for merged positions and over-allocated dividends for unmerged ones.

### Internal Pre-conditions

1. A bidding or reward project is active and at least two TVS NFTs share the same underlying token, each with a non-empty `Allocation` recorded in the project’s `allocations` mapping and mirrored once into `AlignerzVesting::allocationOf`.
2. `A26ZDividendDistributor` (or another consumer) reads per-NFT vesting data from `AlignerzVesting::allocationOf` / `AlignerzVesting::getAllocationOf` rather than directly from the per-project `allocations` mapping.

### External Pre-conditions

1. A TVS holder chooses to consolidate positions by merging one or more NFTs into a single NFT via `AlignerzVesting::mergeTVS`.

### Attack Path

This is a correctness / accounting bug that can be triggered by honest user behavior:

1. A user participates in a bidding project, wins two allocations, and claims two TVS NFTs; `AlignerzVesting` records allocations for both NFT IDs in the project’s `allocations` mapping and copies each allocation into the global `allocationOf` mapping.
2. Before dividends are set up, the user calls `AlignerzVesting::mergeTVS` to merge the second NFT into the first. `mergeTVS` and `_merge` correctly:
   - Append the flows from the second NFT into `biddingProjects[projectId].allocations[mergedNftId]`, minus merge fees.
   - Burn the second NFT.
   - Leave `allocationOf[mergedNftId]` unchanged and `allocationOf[nftIdToMerge]` pointing to a now-burned allocation.
3. Later, the protocol deploys or configures `A26ZDividendDistributor` and calls `setUpTheDividends`, which (after the ABI/interface fixes) relies on `vesting.allocationOf(mergedNftId)` / `vesting.getAllocationOf(mergedNftId)` to compute unclaimed amounts for each existing NFT.
4. For the merged NFT, the distributor sees only the original allocation from `allocationOf[mergedNftId]` (pre-merge flows), while the additional flows that were merged in from the burned NFT are ignored because:
   - They now live only in the project-specific `allocations` mapping; and
   - The burned NFT’s ID fails `safeOwnerOf` and is skipped entirely during dividend setup.
5. The distributor thus underestimates the merged NFT’s unclaimed token amount relative to its true vesting state, reducing the dividend share allocated to the merged position and implicitly boosting the share for other, unmerged TVSs.

### Impact

Merged TVS NFT holders suffer a systematic underpayment of dividends (or other benefits computed off `allocationOf`) whenever `AlignerzVesting::mergeTVS` is used prior to dividend setup:

- Any flows merged into an existing NFT are invisible to `allocationOf`, so `A26ZDividendDistributor::getUnclaimedAmounts` will treat the merged NFT as if it still held only its pre-merge allocation and will not account for the additional flows.
- Burned NFTs are correctly excluded from dividend calculations via `safeOwnerOf`, but their vesting flows are not re-attributed to the surviving NFT in `allocationOf`, causing the total token base used for dividend ratios to diverge from the true vesting state.


### PoC


Before running the POC we have to address a number of pre-existing issues for the scoped contracts to allow the POC to run as intended ( see my other submissions for more details about them):

Please paste this fix in `FeesManager::calculateFeeAndNewAmountForOneTVS`
```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {

        //@audit-info this is my fix
        newAmounts = new uint256[](length);

        for (uint256 i; i < length; i++) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
            
        }
    }
```

FInally, we need to add a  getter for the `allocationOf` mapping to `AlignerzVesting`, as the auto-generated one does not pass dynamic arrays:
```solidity
    function getAllocationOf(
        uint256 nftId
    ) external view returns (Allocation memory) {
        return allocationOf[nftId];
    }
```

Now we can move on to the actual POC:

The following Foundry test shows that after a merge the per-project allocation for the surviving NFT is updated but `AlignerzVesting::allocationOf` (accessed via `getAllocationOf`) remains at its pre-merge value.

Please paste this POC in `AlignerzVestingProtocolTest.t.sol` and run it with `forge clean && forge test --match-test test_POC_merge_keeps_allocationOf_stale -vv`

```solidity

    function test_POC_merge_keeps_allocationOf_stale() public {
        address bidderA = bidders[0];
        address bidderB = bidders[1];
        uint256 vestingPeriod = 90 days;

        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1_000_000,
            "0x0",
            true
        );
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vesting.addUsersToWhitelist(bidders, PROJECT_ID);
        vm.stopPrank();

        address[2] memory participants = [bidderA, bidderB];
        for (uint256 i; i < participants.length; i++) {
            vm.startPrank(participants[i]);
            usdt.approve(address(vesting), BIDDER_USD);
            vesting.placeBid(PROJECT_ID, BIDDER_USD, vestingPeriod);
            vm.stopPrank();
        }

        BidInfo[] memory bids = new BidInfo[](2);
        bids[0] = BidInfo({
            bidder: bidderA,
            amount: BIDDER_USD,
            vestingPeriod: vestingPeriod,
            poolId: 0,
            accepted: true
        });
        bids[1] = BidInfo({
            bidder: bidderB,
            amount: BIDDER_USD,
            vestingPeriod: vestingPeriod,
            poolId: 0,
            accepted: true
        });
        bytes32 poolRoot = generateMerkleProofs(bids, 0);
        bytes32[] memory merkleRoots = new bytes32[](1);
        merkleRoots[0] = poolRoot;

        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, "", merkleRoots, 60 days);

        vm.startPrank(bidderA);
        uint256 mergedNftId = vesting.claimNFT(
            PROJECT_ID,
            0,
            BIDDER_USD,
            bidderProofs[bidderA]
        );
        vm.stopPrank();

        vm.startPrank(bidderB);
        uint256 nftIdToMerge = vesting.claimNFT(
            PROJECT_ID,
            0,
            BIDDER_USD,
            bidderProofs[bidderB]
        );
        vm.stopPrank();

        vm.prank(bidderB);
        nft.safeTransferFrom(bidderB, bidderA, nftIdToMerge);

        AlignerzVesting.Allocation memory beforeMerge = vesting.getAllocationOf(
            mergedNftId
        );

        uint256[] memory projectIdsToMerge = new uint256[](1);
        projectIdsToMerge[0] = PROJECT_ID;
        uint256[] memory nftIdsToMerge = new uint256[](1);
        nftIdsToMerge[0] = nftIdToMerge;

        vm.prank(bidderA);
        vesting.mergeTVS(PROJECT_ID, mergedNftId, projectIdsToMerge, nftIdsToMerge);

        AlignerzVesting.Allocation memory afterMerge = vesting.getAllocationOf(
            mergedNftId
        );

        // after merge, allocation returned by allocationOf is stale !!!
        assertEq(afterMerge.amounts.length, beforeMerge.amounts.length);
        assertEq(afterMerge.amounts[0], beforeMerge.amounts[0]);
    }
```

### Mitigation

Keep the global `allocationOf` mapping synchronized with per-project allocation updates performed by `mergeTVS` and `_merge`:

- After all fees and flow transfers are applied to `mergedTVS` in `AlignerzVesting::mergeTVS`, explicitly write the updated struct back into `allocationOf[mergedNftId]` so that external consumers see the merged flows.
- For each burned `nftId` inside `_merge`, clear or reset `allocationOf[nftId]` to avoid leaving dead allocations that might be accidentally read by other consumers in the future.
- As a pattern, ensure that any future TVS management operations (splits, merges, reassignments) consistently update both the project-specific `allocations` mapping and the global `allocationOf` mapping, or alternatively deprecate `allocationOf` in favor of a single canonical storage layout that all external modules query.


  