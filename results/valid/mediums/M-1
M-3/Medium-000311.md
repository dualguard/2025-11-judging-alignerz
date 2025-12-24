# [000311] Dividend distribution ignores vested/claimed tokens and over-allocates dividends
  
  ## Finding Description
The protocol computes dividend shares for TVS holders based on `allocationOf` data in `AlignerzVesting`, but `allocationOf` is a *snapshot* taken when the NFT is created or split and is **never updated** when vesting claims or merges happen. As a result, `A26ZDividendDistributor` uses stale vesting information and systematically overestimates “unclaimed” amounts for TVSs that have already partially or fully vested.

### Relevant state in `AlignerzVesting`

`AlignerzVesting` maintains the true vesting state per project and NFT in:

[AlignerzVesting.sol#L71-L80](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L71-L80)
```solidity
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
```

With the authoritative storage under:

```solidity
mapping(uint256 => BiddingProject) public biddingProjects;
mapping(uint256 => RewardProject) public rewardProjects;
// each has `mapping(uint256 => Allocation) allocations`
```

In addition, there is a separate mapping exposed as `allocationOf` (renamed to `allocationSnapshot` in the PoC patch), which is populated only at NFT creation or split:

- When distributing remaining reward TVS:

```solidity
rewardProject.allocations[nftId].amounts.push(amount);
...
rewardProject.allocations[nftId].claimedSeconds.push(0);
rewardProject.allocations[nftId].claimedFlows.push(false);
rewardProject.allocations[nftId].token = rewardProject.token;
rewardProject.kolTVSAddresses.pop();
allocationOf[nftId] = rewardProject.allocations[nftId];
```

- When a KOL claims reward TVS:

```solidity
rewardProject.allocations[nftId].amounts.push(amount);
...
rewardProject.allocations[nftId].claimedSeconds.push(0);
rewardProject.allocations[nftId].claimedFlows.push(false);
rewardProject.allocations[nftId].token = rewardProject.token;
allocationOf[nftId] = rewardProject.allocations[nftId];
```

- When a bidding NFT is claimed:

```solidity
biddingProject.allocations[nftId].amounts.push(amount);
biddingProject.allocations[nftId].vestingPeriods.push(bid.vestingPeriod);
biddingProject.allocations[nftId].vestingStartTimes.push(biddingProject.endTime);
biddingProject.allocations[nftId].claimedSeconds.push(0);
biddingProject.allocations[nftId].claimedFlows.push(false);
biddingProject.allocations[nftId].assignedPoolId = poolId;
biddingProject.allocations[nftId].token = biddingProject.token;
NFTBelongsToBiddingProject[nftId] = true;
allocationOf[nftId] = biddingProject.allocations[nftId];
```

- When splitting a TVS:

```solidity
Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
NFTBelongsToBiddingProject[nftId] = isBiddingProject ? true : false;
Allocation storage newAlloc = isBiddingProject
    ? biddingProjects[projectId].allocations[nftId]
    : rewardProjects[projectId].allocations[nftId];
_assignAllocation(newAlloc, alloc);
allocationOf[nftId] = newAlloc;
```

**Crucially**, when actual vesting progress occurs, the protocol only updates the project-level `allocations`, not `allocationOf`:

- In `claimTokens()`:

```solidity
bool isBiddingProject = NFTBelongsToBiddingProject[nftId];
(Allocation storage allocation, IERC20 token) = isBiddingProject ?
    (biddingProjects[projectId].allocations[nftId], biddingProjects[projectId].token) :
    (rewardProjects[projectId].allocations[nftId], rewardProjects[projectId].token);

for (uint256 i; i < nbOfFlows; i++) {
    if (allocation.claimedFlows[i]) { ... continue; }

    (uint256 claimableAmount, uint256 claimableSeconds) =
        getClaimableAmountAndSeconds(allocation, i);

    allocation.claimedSeconds[i] += claimableSeconds;
    if (allocation.claimedSeconds[i] >= allocation.vestingPeriods[i]) {
        flowsClaimed++;
        allocation.claimedFlows[i] = true;
    }
    ...
}
if (flowsClaimed == nbOfFlows) {
    nftContract.burn(nftId);
    allocation.isClaimed = true;
}
token.safeTransfer(msg.sender, claimableAmounts);
```

Note that `allocationOf[nftId]` is **never updated** here, even though `allocation.claimedSeconds[i]`, `allocation.claimedFlows[i]`, and `allocation.isClaimed` change.

Similarly, merges adjust the underlying `Allocation` but not `allocationOf`:

```solidity
function _merge(Allocation storage mergedTVS, uint256 projectId, uint256 nftId, IERC20 token) internal returns (uint256 feeAmount) {
    ...
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
    nftContract.burn(nftId);
}
```

Again, no write to `allocationOf[mergedNftId]` occurs.

### How `A26ZDividendDistributor` misuses `allocationOf`
`A26ZDividendDistributor` uses `allocationOf` as the basis for computing “unclaimed amounts” per NFT:

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    IAlignerzVesting.Allocation memory alloc = vesting.allocationOf(nftId);
    if (address(token) == address(alloc.token)) return 0;

    uint256 len = alloc.amounts.length;
    for (uint256 i; i < len; ) {
        if (alloc.claimedFlows[i]) {
            unchecked { ++i; }
            continue;
        }

        if (alloc.claimedSeconds[i] == 0) {
            amount += alloc.amounts[i];
            unchecked { ++i; }
            continue;
        }

        uint256 claimedAmount = alloc.claimedSeconds[i] * alloc.amounts[i] / alloc.vestingPeriods[i];
        uint256 unclaimedAmount = alloc.amounts[i] - claimedAmount;
        amount += unclaimedAmount;

        unchecked { ++i; }
    }

    unclaimedAmountsIn[nftId] = amount;
}
```

Since `alloc` is loaded from `vesting.allocationOf(nftId)`, which is a **static snapshot** taken at NFT creation/split time, all of the following are stale after any vesting activity:

- `alloc.claimedSeconds[i]` remains `0` even if half or all of the flow has been claimed.
- `alloc.claimedFlows[i]` remains `false` even if the entire flow was fully vested and marked claimed in the project-level `allocations`.
- `alloc.isClaimed` is not updated when all flows are claimed and the NFT is burned.

The worst-case scenario is:

1. A TVS NFT is created with some `amounts[i]` and `vestingPeriods[i]`.
2. The user calls `claimTokens()` multiple times and fully claims one or more flows.
3. The *authoritative* `Allocation` under `biddingProjects[projectId].allocations[nftId]` or `rewardProjects[projectId].allocations[nftId]` now has `claimedSeconds[i] == vestingPeriods[i]` and `claimedFlows[i] == true`.
4. `allocationOf[nftId]` still shows `claimedSeconds[i] == 0` and `claimedFlows[i] == false`.
5. When the treasury later funds `A26ZDividendDistributor` and the owner calls `setAmounts()` / `setDividends()`, the distributor uses this stale snapshot:
   - For flows with `claimedSeconds[i] == 0`, it treats the entire `amounts[i]` as unclaimed.
   - For flows with partial claims, it underestimates `claimedAmount` since `claimedSeconds[i]` remains lower than the true claimed seconds.

This systematically overestimates the “unclaimed” amount for any TVS that has previously claimed tokens, and thus **over-allocates dividends** to those TVSs at the expense of others who have not claimed yet.

The PoC test `test_DividendDistributor_UsesStaleAllocationSnapshot()` in the _Proof of Concept_ section demonstrates this by:

- Creating a bidding project and pool.
- Whitelisting a bidder and having them place a bid with `vestingPeriodSec = 100`.
- Finalizing the project with a single-leaf Merkle root and letting the bidder claim an NFT (`nftId`).
- Reading `allocationOf(nftId)` before any claim and confirming `claimedSeconds[0] == 0`.
- Warping half the vesting period and calling `claimTokens(PROJECT_ID, nftId)`, which transfers half the tokens (`amount / 2`) to the user.
- Reading `allocationOf(nftId)` again and confirming `claimedSeconds[0]` is still `0` (snapshot not updated).
- Deploying `A26ZDividendDistributor` and calling `getUnclaimedAmounts(nftId)`:
  - The function treats the full `amount` as unclaimed because it uses the stale snapshot.
  - The test checks that `unclaimedFromDistributor == amount`, while the correct unclaimed amount (`expectedUnclaimedAmount`) is strictly smaller.

This PoC confirms that dividend calculations are based on obsolete vesting data and do **not** reflect actual claimed amounts.


## Impact
Medium. Funds are not directly stolen from the protocol contract, but dividend distribution between users is economically incorrect and can result in significant over- or under-payment to specific TVS holders.

## Likelihood
Medium. The protocol is explicitly designed to allow claims before dividends are set up; if `A26ZDividendDistributor` is deployed and used in production after any vesting activity, the divergence between real vesting state and the snapshot will occur in normal operation.

## Proof of Concept

To get the PoC to execute end-to-end, I locally:
- Fixed the ABI mismatch of allocationOf so it returns the full Allocation struct. (my 4th submitted bug)
- Fixed the infinite loop in getUnclaimedAmounts() (missing i++ on continue paths). (my 2nd submitted bug)

Now add the PoC test in `AlignerzVestingProtocolTest.t.sol`:

1. Import these:
```bash
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
```

2. PoC test:

```solidity
    function test_DividendDistributor_UsesStaleAllocationSnapshot() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, bytes32(0), true);
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, false);
        vesting.addUsersToWhitelist(bidders, PROJECT_ID);
        vm.stopPrank();

        address bidder = bidders[0];

        vm.startPrank(bidder);
        usdt.approve(address(vesting), BIDDER_USD);
        uint256 vestingPeriodSec = 100;
        vesting.placeBid(PROJECT_ID, BIDDER_USD, vestingPeriodSec);
        vm.stopPrank();

        BidInfo[] memory bids = new BidInfo[](1);
        bids[0] = BidInfo({
            bidder: bidder,
            amount: BIDDER_USD,
            vestingPeriod: vestingPeriodSec,
            poolId: 0,
            accepted: true
        });

        // For a single-leaf Merkle tree, the root is just the leaf and the proof is empty
        bytes32 leaf = getLeaf(bidder, BIDDER_USD, PROJECT_ID, 0);
        bytes32[] memory proof = new bytes32[](0);
        bidderProofs[bidder] = proof;

        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = leaf;

        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 7 days);

        vm.prank(bidder);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[bidder]);

        IAlignerzVesting vestingView = IAlignerzVesting(address(vesting));
        IAlignerzVesting.Allocation memory snapshotBefore = vestingView.allocationOf(nftId);
        assertEq(snapshotBefore.claimedSeconds.length, 1);
        assertEq(snapshotBefore.claimedSeconds[0], 0);

        vm.warp(block.timestamp + vestingPeriodSec / 2);
        uint256 tokenBalanceBefore = token.balanceOf(bidder);

        vm.prank(bidder);
        vesting.claimTokens(PROJECT_ID, nftId);

        uint256 tokenBalanceAfter = token.balanceOf(bidder);
        assertTrue(tokenBalanceAfter > tokenBalanceBefore, "Partial claim should transfer tokens");

        IAlignerzVesting.Allocation memory snapshotAfter = vestingView.allocationOf(nftId);
        assertEq(snapshotAfter.claimedSeconds[0], 0, "allocationOf snapshot is not updated after claims");

        uint256 expectedClaimedSeconds = vestingPeriodSec / 2;
        uint256 amount = snapshotAfter.amounts[0];
        uint256 vestingPeriodFromSnapshot = snapshotAfter.vestingPeriods[0];
        uint256 expectedClaimedAmount = amount * expectedClaimedSeconds / vestingPeriodFromSnapshot;
        uint256 expectedUnclaimedAmount = amount - expectedClaimedAmount;

        A26ZDividendDistributor distributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(usdt),
            block.timestamp,
            30 days,
            address(usdt)
        );

        uint256 unclaimedFromDistributor = distributor.getUnclaimedAmounts(nftId);

        assertEq(unclaimedFromDistributor, amount, "Distributor treats full allocation as unclaimed");
        assertLt(expectedUnclaimedAmount, unclaimedFromDistributor, "Expected unclaimed amount should be lower after partial claim");
    }

```

**How to run the PoC**

From the `protocol` directory:

```bash
forge test --mt test_DividendDistributor_UsesStaleAllocationSnapshot
```

This will run only the PoC test and display the logs and asserts that demonstrate the stale `allocationOf` snapshot and incorrect dividend unclaimed amount.

## Recommendation
The core problem is that `allocationOf[nftId]` (or its snapshot equivalent) is not updated whenever the underlying `Allocation` is mutated during claims and merges. The simplest fix is to **keep `allocationOf` in sync** with the real `allocations` by writing back the updated `Allocation` whenever vesting state changes.


  