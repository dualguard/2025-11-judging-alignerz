# [000002] Splitting TVSs will burn vesting allocations for TVS NFT holders and they will lose funds
  
  ### Summary

The repeated use of a mutated storage allocation as the base for each split in [`AlignerzVesting::splitTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054) will cause a silent loss of vested tokens for TVS NFT holders as any user who splits a TVS into multiple NFTs will have each subsequent percentage applied to an already reduced amount, so the sum of post-split allocations is strictly less than the pre-split allocation.

### Root Cause

In `AlignerzVesting::splitTVS` the function first computes post-fee amounts and writes them back to the original NFT’s `Allocation` in storage, then uses that same storage reference as the base for every per-output split, including the original NFT:

```solidity
function splitTVS(
    uint256 projectId,
    uint256[] calldata percentages,
    uint256 splitNftId
) external returns (uint256, uint256[] memory) {
    // ...
    bool isBiddingProject = NFTBelongsToBiddingProject[splitNftId];
    // in-storage allocation is loaded
    (Allocation storage allocation, IERC20 token) = isBiddingProject
        ? (
            biddingProjects[projectId].allocations[splitNftId],
            biddingProjects[projectId].token
        )
        : (
            rewardProjects[projectId].allocations[splitNftId],
            rewardProjects[projectId].token
        );

    // ...
    for (uint256 i; i < nbOfTVS; ) {

        // ...

@>      Allocation memory alloc =
@>          _computeSplitArrays(allocation, percentage, nbOfFlows);
        // modified allocation is stored directly in storage, so next calls of _computeSplitArrays will use a decreased amounts!!!
        Allocation storage newAlloc = isBiddingProject
            ? biddingProjects[projectId].allocations[nftId]
            : rewardProjects[projectId].allocations[nftId];
@>      _assignAllocation(newAlloc, alloc);

        // ...
    }

}
```

The key points are:

- `allocation` is a storage reference to the original NFT’s `Allocation` in either `biddingProjects[projectId].allocations[splitNftId]` or `rewardProjects[projectId].allocations[splitNftId]`.
- `_computeSplitArrays` reads from that same storage allocation to build each per-output `Allocation`:

```solidity
function _computeSplitArrays(
    Allocation storage allocation,
    uint256 percentage,
    uint256 nbOfFlows
) internal view returns (Allocation memory alloc) {
    uint256[] memory baseAmounts = allocation.amounts;
    // ...
    alloc.amounts = new uint256[](nbOfFlows);
    // ...
    for (uint256 j; j < nbOfFlows; ) {
@>      alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
        // ...
        unchecked {
            ++j;
        }
    }
}
```

- For `i == 0`, `nftId == splitNftId`, so `newAlloc` is the same storage slot as `allocation`. `_assignAllocation(newAlloc, alloc)` therefore overwrites the original NFT’s `Allocation` with the first split’s amounts.
- For `i > 0`, `_computeSplitArrays` still reads from the `allocation` storage that was already overwritten in the previous iteration, so each subsequent `percentage` is applied to an allocation that has already been scaled by previous percentages.

Concretely, if a user splits a TVS into two NFTs with `[5_000, 5_000]` (intended 50/50 split, ignoring fees):

- Before the loop, `allocation.amounts` holds the full post-fee amount `A`.
- Iteration `i = 0`: `_computeSplitArrays(allocation, 5_000)` computes `alloc0 = 0.5 * A`, and `_assignAllocation` writes this back into the original NFT’s storage.
- Iteration `i = 1`: `_computeSplitArrays(allocation, 5_000)` now sees `allocation.amounts == alloc0 == 0.5 * A` and computes `alloc1 = 0.5 * (0.5 * A) = 0.25 * A`.

The total allocation after the split is `0.5 * A + 0.25 * A = 0.75 * A`, so 25% of the user’s vesting allocation silently disappears from all `Allocation` records and becomes permanently unclaimable. With more outputs or different percentages, the discrepancy between the pre-split and post-split totals can be even larger, but the user has no visibility into the lost portion beyond reduced claimable amounts.


### Internal Pre-conditions

1. A bidding or reward project is live and a user has a TVS NFT with a non-empty `Allocation` (i.e. `allocation.amounts.length > 0`) recorded in `AlignerzVesting`.

### External Pre-conditions

1. The user chooses to split their TVS NFT into more than one output by calling `AlignerzVesting::splitTVS` with `percentages.length > 1`.

### Attack Path

1. A user participates in a bidding project, wins an allocation, and claims a TVS NFT; `AlignerzVesting` records an `Allocation` entry for that `nftId`.
2. The user decides to reorganize their vesting and calls `AlignerzVesting::splitTVS(projectId, [5_000, 5_000], nftId)` to split the position into two 50/50 NFTs.
3. Inside `splitTVS`, the contract computes post-fee amounts and writes them back to the original `allocation.amounts`, then starts the loop over `percentages`.
4. On the first iteration, the contract computes `alloc0` from the full allocation and overwrites `allocation` (the original NFT) with that first share via `_assignAllocation`.
5. On the second iteration, `_computeSplitArrays` reads from the already-overwritten `allocation` and computes `alloc1` as a percentage of the reduced amount rather than the original, resulting in a much smaller second share.
6. After the loop, the sum of `allocation.amounts` for the original and new NFTs is strictly less than the pre-split amount; the missing portion is never attributed to any NFT and cannot be claimed by the user.

### Impact

The affected party is any TVS NFT holder who uses `splitTVS` to reorganize their vesting position:

- For a simple 50/50 split into two NFTs, the user permanently loses 25% of their vesting allocation (post-fee) because the second output is computed from an already-halved base.
- For splits into more than two NFTs or with uneven percentages, the mismatch between the intended percentages and the actual allocated amounts can be even more severe, with the missing portion silently locked in the contract and never credited to any NFT.
- This is a direct, irreversible loss of user funds triggered by normal use of a core feature (TVS splitting), which should be safe and value-conserving.

### PoC

Before running the actual POC there are a number of steps that must be done, as the current scoped contracts have other bugs that revert any call to `splitTVS`.

Please paste this fixed function in `AlignerzVesting::_computeSplitArrays`
```solidity
    // Needed for this PoC: initialize alloc arrays in _computeSplitArrays so
    // splitTVS does not revert due to uninitialized memory arrays. This fix
    // addresses a separate issue (uninitialized arrays in the split helper).
        function _computeSplitArrays(
        Allocation storage allocation,
        uint256 percentage,
        uint256 nbOfFlows
    ) internal view returns (Allocation memory alloc) {
        uint256[] memory baseAmounts = allocation.amounts;
        uint256[] memory baseVestings = allocation.vestingPeriods;
        uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
        uint256[] memory baseClaimed = allocation.claimedSeconds;
        bool[] memory baseClaimedFlows = allocation.claimedFlows;
        alloc.assignedPoolId = allocation.assignedPoolId;
        alloc.token = allocation.token;

        // @audit-info these are my added fixes
        alloc.amounts = new uint256[](nbOfFlows);
        alloc.vestingPeriods = new uint256[](nbOfFlows);
        alloc.vestingStartTimes = new uint256[](nbOfFlows);
        alloc.claimedSeconds = new uint256[](nbOfFlows);
        alloc.claimedFlows = new bool[](nbOfFlows);

        for (uint256 j; j < nbOfFlows; ) {
            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
            alloc.vestingPeriods[j] = baseVestings[j];
            alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
            alloc.claimedSeconds[j] = baseClaimed[j];
            alloc.claimedFlows[j] = baseClaimedFlows[j];
            unchecked {
                ++j;
            }
        }
    }
```

And paste also this other fix in `FeesManager::calculateFeeAndNewAmountForOneTVS`
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

Now we can paste and run the actual POC

The following Foundry test (add to `AlignerzVestingProtocolTest.t.sol`) demonstrates that a seemingly fair 50/50 split into two NFTs results in a total post-split allocation that is strictly less than the pre-split allocation recorded for the original TVS:

```solidity

        function _setupAndFinalizeProjectWithBidder(address bidder) internal {


        // setup bidding project
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
        vm.startPrank(bidder);
        usdt.approve(address(vesting), BIDDER_USD);
        uint256 vestingPeriod = 90 days;
        vesting.placeBid(PROJECT_ID, BIDDER_USD, vestingPeriod);
        vm.stopPrank();

        BidInfo memory bid = BidInfo({
            bidder: bidder,
            amount: BIDDER_USD,
            vestingPeriod: vestingPeriod,
            poolId: 0,
            accepted: true
        });
        BidInfo[] memory bids = new BidInfo[](2);
        bids[0] = bid;
        bids[1] = bid;
        bytes32 poolRoot = generateMerkleProofs(bids, 0);
        bytes32[] memory merkleRoots = new bytes32[](1);
        merkleRoots[0] = poolRoot;

        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, "", merkleRoots, 60 days);

    }

    function test_POC_split_reduces_total() public {

        address victim = bidders[0];

        _setupAndFinalizeProjectWithBidder(victim);

        vm.startPrank(victim);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[victim]);
        assertEq(victim, nft.ownerOf(nftId));
        uint256[] memory percentages = new uint256[](2);
        // 50% each nft
        percentages[0] = 5_000;
        percentages[1] = 5_000;


        uint preSplitAmount = vesting.getAllocationOf(nftId).amounts[0];
        (, uint256[] memory newNftIds) = vesting.splitTVS(PROJECT_ID, percentages, nftId);

        uint postSplitAmount1 = vesting.getAllocationOf(nftId).amounts[0];
        uint postSplitAmount2 = vesting.getAllocationOf(newNftIds[0]).amounts[0];

        assertLt(postSplitAmount1 + postSplitAmount2, preSplitAmount);

        console.log("Pre-split amount:", preSplitAmount / 1e18);
        console.log("Post-split amount 1:", postSplitAmount1 / 1e18);
        console.log("Post-split amount 2:", postSplitAmount2 / 1e18);

        vm.stopPrank();

    }
```

please paste this POC in `AlignerzVestingProtocolTest.t.sol` and run it with `forge clean && forge test --match-test test_POC_split_reduces_total -vv`

### Mitigation

- Ensure that each split output is computed from the original pre-split allocation, not from a storage slot that is being mutated inside the loop. For example, copy the base `Allocation` (or at least its `amounts` array) to memory once before the loop and have `_computeSplitArrays` use that immutable base rather than `allocation` storage.
- Avoid applying `_assignAllocation` to the original NFT (`splitNftId`) until all per-output allocations have been computed from the immutable base; then write each final `Allocation` into its target NFT storage exactly once.
- Add invariant checks (in tests or internal asserts) to verify that the sum of all post-split `amounts` across the original and new NFTs equals the pre-split amount (minus any explicit fees), so that future refactors cannot reintroduce silent value loss.


  