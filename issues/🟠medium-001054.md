# [001054] [M] Stale allocationOf after merge (and burn) misreports TVS data and keeps ghost entries
  
  ### Summary
After merging one TVS (NFT) into another, the canonical allocation for the target NFT is updated, and the source NFT is typically burned. However, the mirror mapping `allocationOf` is not synchronized for the target NFT, and the burned source NFT’s mirror entry is not deleted. Any consumer that reads `allocationOf` (e.g., via `getAllocation`) will observe:
- The target NFT still showing its pre‑merge flows/amounts (stale), and
- The burned NFT still present in `allocationOf` (ghost entry).

This leads to incorrect downstream calculations (e.g., unclaimed computation) and confusing state where the on-chain NFT count differs from the number of entries that appear to be “live” in `allocationOf`.

### Code context
- Merge path updates only the canonical allocation, not the mirror:
```1045:1069:protocol/src/contracts/vesting/AlignerzVesting.sol
function mergeTVS(uint256 projectId, uint256 mergedNftId, uint256[] calldata projectIds, uint256[] calldata nftIds) external returns(uint256) {
    ...
    (Allocation storage mergedTVS, IERC20 token) = isBiddingProject ?
        (biddingProjects[projectId].allocations[mergedNftId], biddingProjects[projectId].token) :
        (rewardProjects[projectId].allocations[mergedNftId], rewardProjects[projectId].token);
    uint256[] memory amounts = mergedTVS.amounts;
    uint256 nbOfFlows = mergedTVS.amounts.length;
    (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
    mergedTVS.amounts = newAmounts;
    ... // merge in nftIds[i] into mergedNftId and burn sources
    emit TVSsMerged(...);
    return mergedNftId;
}
```
- Split path shows how `allocationOf` is written for new NFTs, while merge path lacks a corresponding sync/delete:
```1131:1136:protocol/src/contracts/vesting/AlignerzVesting.sol
Allocation storage newAlloc = isBiddingProject ? biddingProjects[projectId].allocations[nftId] : rewardProjects[projectId].allocations[nftId];
_assignAllocation(newAlloc, alloc);
allocationOf[nftId] = newAlloc; // mirror is written here for splits/new NFTs
```
- Older distributor logic used `allocationOf` via `getAllocation` (and even now any consumer using it will read the mirror):
```148:170:protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    (
        uint256[] memory amounts,
        uint256[] memory vestingPeriods,
        ,
        uint256[] memory claimedSeconds,
        bool[] memory claimedFlows,
        ,
        IERC20 allocToken,
        /*assignedPoolId*/
    ) = vesting.getAllocation(nftId); // reads mirror allocation
    if (address(token) == address(allocToken)) return 0;
    ...
}
```

### Root cause
Two independent storage locations are used for the same logical data:
- Canonical per‑project storage: `biddingProjects[...].allocations[nftId]` or `rewardProjects[...].allocations[nftId]` (updated on merge, claim, split).
- Mirror: `allocationOf[nftId]` (set at mint/split; not updated on merge and not deleted on burn).

Because merge updates only the canonical allocation and burning the source does not `delete allocationOf[sourceNftId]`, the mirror diverges (stale) and retains ghost entries.

### Impact
The imact includes following parts
- Incorrect downstream calculations for any module reading `allocationOf` (e.g., distributor sees outdated flows/amounts; can over/under‑allocate dividends or compute wrong unclaimed values).
- Ghost entries: burned NFTs still appear to have allocations, confusing off‑chain tools and auditors.
- Data inconsistency: number of real NFTs (ERC721A) differs from the set of NFTs visible in `allocationOf`.

### PoC (tests)
![IMPORTANT](https://img.shields.io/badge/IMPORTANT-red)
> **⚠️ Note:** In order to run this test you should implement properly the mitigation steps from [this](https://github.com/dualguard/2025-11-alignerz-konstantinvelev/issues/2) and also [this](https://github.com/dualguard/2025-11-alignerz-konstantinvelev/issues/6) one

Please add this test in the test file.

```solidity
    function test_MirrorAllocation_StaleAfterMerge_MisreportsClaimable() public {
        // PoC: After merge, canonical allocation for target NFT changes,
        // but mirror (used by getAllocation) is not synchronized -> mismatch with actual claim.

        // Ensure merge fee is 0 for determinism
        vm.prank(projectCreator);
        vesting.setFees(0, 0, 0, 0);

        // Launch reward project and mint two TVSs to two KOLs, then transfer to kolA to allow merge
        address kolA = bidders[16];
        address kolB = bidders[17];
        uint256 tvsAmount = 100 ether;
        uint256 vestingPeriod = 90 days;

        vm.startPrank(projectCreator);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 7 days);
        address[] memory kols = new address[](2);
        uint256[] memory amounts = new uint256[](2);
        kols[0] = kolA;
        kols[1] = kolB;
        amounts[0] = tvsAmount;
        amounts[1] = tvsAmount;
        vesting.setTVSAllocation(0, tvsAmount * 2, vestingPeriod, kols, amounts);
        vm.stopPrank();

        vm.prank(kolA);
        vesting.claimRewardTVS(0); // nftId 1
        vm.prank(kolB);
        vesting.claimRewardTVS(0); // nftId 2

        // Transfer id 2 to kolA so kolA owns both -> can merge id2 into id1
        vm.prank(kolB);
        nft.approve(kolA, 2);
        vm.prank(kolB);
        nft.transferFrom(kolB, kolA, 2);

        // Capture mirror length before merge
        (
            uint256[] memory preAmounts,
            ,
            ,
            ,
            ,
            ,
            ,

        ) = vesting.getAllocation(1);
        uint256 preLen = preAmounts.length;
        console2.log("before merge, reported amounts.length for id=1 =", preLen);
        // Also capture source NFT mirror length to show two separate mirror entries pre-merge
        (
            uint256[] memory preSrcAmounts,
            ,
            ,
            ,
            ,
            ,
            ,

        ) = vesting.getAllocation(2);
        console2.log("before merge, reported amounts.length for id=2 =", preSrcAmounts.length);

        // Merge: projectId=0, mergedNftId=1, merge nftIds=[2]
        console2.log("Merging nftId=2 into nftId=1");
        uint256[] memory projIds = new uint256[](1);
        projIds[0] = 0;
        uint256[] memory toMerge = new uint256[](1);
        toMerge[0] = 2;
        vm.prank(kolA);
        vesting.mergeTVS(0, 1, projIds, toMerge);

        // After merge: compare the reported sum(amounts) for target NFT with the expected canonical sum.
        // Expected canonical sum after merge = preSum(target) + preSum(source)
        uint256 expectedSumAfterMerge = 0;
        for (uint256 i = 0; i < preAmounts.length; i++) expectedSumAfterMerge += preAmounts[i];
        for (uint256 i = 0; i < preSrcAmounts.length; i++) expectedSumAfterMerge += preSrcAmounts[i];

        // Read reported amounts (from getAllocation) for target id=1 AFTER merge
        (uint256[] memory mAmounts,,,,,,,) = vesting.getAllocation(1);
        console2.log("after merge, reported amounts.length for id=1 =", mAmounts.length);
        uint256 reportedSumAfterMerge = 0;
        for (uint256 i = 0; i < mAmounts.length; i++) reportedSumAfterMerge += mAmounts[i];
        // total TVS reported vs expected merged sum are compared below via assertions
        // Read reported amounts for burned source id to show it's still present (stale)
        (
            uint256[] memory postSrcAmounts,
            ,
            ,
            ,
            ,
            ,
            ,

        ) = vesting.getAllocation(2);
        console2.log("after merge, reported amounts.length for burned id=2 =", postSrcAmounts.length);
        console2.log("We can see that after the merege we are still seeing the same amount of flows for the target merge-into NFT");
        console2.log("preAmounts.length =", preAmounts.length);
        console2.log("mAmounts.length =", mAmounts.length);
        assertEq(preAmounts.length, mAmounts.length);

        // Additionally: mirror flow count should remain the same if it wasn't synchronized
        assertEq(mAmounts.length, preLen, "Mirror amounts length changed but should remain stale after merge");
        // And mirror still holds data for burned NFT (id=2)
        assertGt(postSrcAmounts.length, 0, "Mirror for burned NFT should still have stale data");
    }
```

Test Results:

```solidity
  npm warn exec The following package was not found and will be installed: @openzeppelin/upgrades-core@1.44.2

  before merge, reported amounts.length for id=1 = 1
  before merge, reported amounts.length for id=2 = 1
  Merging nftId=2 into nftId=1
  after merge, reported amounts.length for id=1 = 1
  after merge, reported amounts.length for burned id=2 = 1
  We can see that after the merege we are still seeing the same amount of flows for the target merge-into NFT
  preAmounts.length = 1
  mAmounts.length = 1

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.77s (2.40ms CPU time)

Ran 1 test suite in 5.79s (5.77s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Recommended mitigations

I would recommend to sync the status properly when we merge.
Pseudo fix would look like
```diff
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
        mergedTVS.amounts = newAmounts;

        uint256 nbOfNFTs = nftIds.length;
        require(nbOfNFTs > 0, Not_Enough_TVS_To_Merge());
        require(nbOfNFTs == projectIds.length, Array_Lengths_Must_Match());

        for (uint256 i; i < nbOfNFTs; i++) {
            feeAmount += _merge(mergedTVS, projectIds[i], nftIds[i], token);
+            delete allocationOf[nftIds[i]];
        }
+        allocationOf[mergedNftId] = mergedTVS // this is passed as storage variable so it would be updated
        token.safeTransfer(treasury, feeAmount);
        emit TVSsMerged(projectId, isBiddingProject, nftIds, mergedNftId, mergedTVS.amounts, mergedTVS.vestingPeriods, mergedTVS.vestingStartTimes, mergedTVS.claimedSeconds, mergedTVS.claimedFlows);
        return mergedNftId;
    }
```
  