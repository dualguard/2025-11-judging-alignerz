# [000710] Misindexed reward lists strand unclaimed allocations
  
  
## summary
setTVSAllocation() and setStablecoinAllocation() store the loop counter as the index for every address, corrupting kol…Addresses bookkeeping and causing earlier recipients to disappear from distribution arrays once later batches claim, leaving their funds permanently locked.

## Finding Description  
Each allocation setter pushes addresses into the tracking arrays but records rewardProject.kolTVSIndexOf[kol] = i (the per-call loop counter) instead of the actual index inside the growing array:

[affected function code
](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L464-L471)
```solidity
for (uint256 i = 0; i < length; i++) {
    address kol = kolTVS[i];
    rewardProject.kolTVSAddresses.push(kol);
    uint256 amount = TVSamounts[i];
    rewardProject.kolTVSRewards[kol] = amount;
    rewardProject.kolTVSIndexOf[kol] = i; // wrong if function called multiple times
    ...
}
```

When _claimRewardTVS() executes, it retrieves index = rewardProject.kolTVSIndexOf[kol] and swap-pops the address at that slot. Because the stored index refers to the original loop position, not the actual array slot after earlier pushes, the function removes the wrong address from kolTVSAddresses. The real claimant often stays in the array with a stale index, while some other address (possibly still unclaimed) vanishes. Post-deadline cleanups (distributeRemainingRewardTVS() and distributeRemainingStablecoinAllocation()) iterate over the arrays to mint NFTs or transfer funds to whoever remains. Anyone removed prematurely is never processed, so their allocation stays trapped in the contract forever.

Highest-impact scenario: the owner runs setTVSAllocation() twice (batch A, then B). A member of batch B claims first; the faulty index points to batch A’s slot, so _claimRewardTVS() removes A from kolTVSAddresses even though A never claimed. When the claim window passes, the owner calls distributeRemainingRewardTVS() to finish payouts, but A is no longer in the list, so their tokens are stuck without any way to recover them.
**  Attack Path **
1. Owner configures multiple allocation batches (common for iterative KOL onboarding).
2. A later batch claimant executes claimRewardTVS().
3. The swap-pop removes the wrong address, corrupting kolTVSAddresses.
4. After the deadline, distributeRemainingRewardTVS() or manual auditing shows missing entries, and the real unclaimed addresses can never be reached because the contract no longer tracks them.

**Highest-impact scenario:** Early KOLs (or stablecoin recipients) who miss the deadline lose allocations permanently after other batches claim, despite the protocol attempting to distribute leftovers; tokens remain locked in the contract, impacting treasury accounting and potentially leading to disputes.

## Impact  
High – corrupted bookkeeping can permanently lock legitimate allocations, preventing both claims and owner-enforced distributions and effectively burning treasury tokens/stablecoins.
## poc 

run 
```c
forge clean && forge build && forge test --mt test_TVSAllocationIndexingStrandsEarlyRecipients -vvv
```
```solidity
    function test_TVSAllocationIndexingStrandsEarlyRecipients() public {
        address kolEarly = bidders[0];
        address kolLate = bidders[1];

        vm.startPrank(projectCreator);
        uint256 startTime = block.timestamp;
        uint256 claimWindow = 3 days;
        vesting.launchRewardProject(address(token), address(usdt), startTime, claimWindow);

        address[] memory firstBatchAddresses = new address[](1);
        firstBatchAddresses[0] = kolEarly;
        uint256[] memory firstBatchAmounts = new uint256[](1);
        firstBatchAmounts[0] = 100 ether;
        vesting.setTVSAllocation(PROJECT_ID, 100 ether, 90 days, firstBatchAddresses, firstBatchAmounts);

        address[] memory secondBatchAddresses = new address[](1);
        secondBatchAddresses[0] = kolLate;
        uint256[] memory secondBatchAmounts = new uint256[](1);
        secondBatchAmounts[0] = 200 ether;
        vesting.setTVSAllocation(PROJECT_ID, 200 ether, 90 days, secondBatchAddresses, secondBatchAmounts);
        vm.stopPrank();

        vm.prank(kolLate);
        vesting.claimRewardTVS(PROJECT_ID);

        assertEq(nft.balanceOf(kolLate), 1, "late KOL should hold exactly one NFT after claiming");
        assertEq(nft.balanceOf(kolEarly), 0, "early KOL has not claimed yet");
        assertEq(nft.getTotalMinted(), 1, "only one NFT should be minted so far");

        vm.warp(startTime + claimWindow + 1);

        vm.prank(projectCreator);
        vesting.distributeRemainingRewardTVS(PROJECT_ID);

        uint256 mintedAfterDistribution = nft.getTotalMinted();
        assertEq(mintedAfterDistribution, 2, "distribution minted exactly one additional NFT");
        assertEq(nft.balanceOf(kolLate), 2, "late KOL is minted twice while already claimed");
        assertEq(nft.balanceOf(kolEarly), 0, "early KOL disappeared from the tracking array");

        assertEq(nft.ownerOf(mintedAfterDistribution), kolLate, "distribution re-mints the later KOL");

        vm.expectRevert(AlignerzVesting.Deadline_Has_Passed.selector);
        vm.prank(kolEarly);
        vesting.claimRewardTVS(PROJECT_ID);
    }
```
## Recommendation  
Store the actual array index when pushing and validate before swap-pop. For example:

```diff
    rewardProject.kolTVSAddresses.push(kol);
-   rewardProject.kolTVSIndexOf[kol] = i;
+   rewardProject.kolTVSIndexOf[kol] = rewardProject.kolTVSAddresses.length - 1;
```

Apply the same fix to the stablecoin trackers, and optionally check that kolTVSAddresses[index] == kol before removing to detect earlier corruption.

  