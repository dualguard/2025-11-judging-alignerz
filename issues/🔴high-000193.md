# [000193] `calculateFeeAndNewAmountForOneTVS` bricks split/merge flows
  
  ### Summary

 Failing to allocate the `newAmounts` array in `protocol/src/contracts/vesting/feesManager/FeesManager.sol:169` will cause a protocol-wide liveness failure for all vesting users as any caller will revert the split/merge fee helper before a single amount is processed.

### Root Cause

- In `protocol/src/contracts/vesting/feesManager/FeesManager.sol:169` the `newAmounts` return array is declared but never assigned, so `newAmounts[i] = ...` is an out-of-bounds write that panics on the first loop iteration..

### Internal Pre-conditions

1. Project owner launches a reward or bidding project and allocates at least one vesting flow to a user so `amounts.length > 0`.
2. The vesting contract is configured with non-zero split or merge fee rates so `calculateFeeAndNewAmountForOneTVS` is invoked inside `splitTVS`/`mergeTVS`.

### External Pre-conditions

None

### Attack Path

 1. A user who owns a TVS NFT (reward or bidding) calls `splitTVS` or `mergeTVS`.
2. The public function calls `calculateFeeAndNewAmountForOneTVS` with the flow amounts array and its length.
3. The helper reverts with `Panic(0x32)` on the first iteration because `newAmounts[0]` is accessed without allocating the array, cancelling the user operation.


### Impact

 The users cannot split TVSs or merge TVSs. All flows that rely on fee deduction before adjusting vesting schedules are permanently broken (protocol-wide liveness failure).

### PoC

```solidity
 function test_PoC_splitTVSRevertsDueToUnallocatedFeeArray() public {
        address kol = bidders[0];
        uint256 rewardProjectId = vesting.rewardProjectCount();
        uint256 allocationAmount = 1_000 ether;
        uint256 vestingPeriod = 30 days;
        uint256 claimWindow = 60 days;
        uint256 startTime = block.timestamp;

        vm.startPrank(projectCreator);
        vesting.launchRewardProject(address(token), address(usdt), startTime, claimWindow);
        address[] memory recipients = new address[](1);
        recipients[0] = kol;
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = allocationAmount;
        vesting.setTVSAllocation(rewardProjectId, allocationAmount, vestingPeriod, recipients, amounts);
        vm.stopPrank();

        vm.prank(kol);
        vesting.claimRewardTVS(rewardProjectId);
        uint256 nftId = nft.getTotalMinted();

        uint256[] memory percentages = new uint256[](1);
        percentages[0] = vesting.BASIS_POINT();

        vm.prank(kol);
        vm.expectRevert(stdError.indexOOBError);
        vesting.splitTVS(rewardProjectId, percentages, nftId);
    }


```

├─ [16967] ERC1967Proxy::fallback(0, [10000 [1e4]], 1)
    │   ├─ [16211] AlignerzVesting::splitTVS(0, [10000 [1e4]], 1) [delegatecall]
    │   │   ├─ [4318] AlignerzNFT::extOwnerOf(1) [staticcall]
    │   │   │   └─ ← [Return] bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083]
    │   │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    └─ ← [Return]

### Mitigation

Initialize `new uint256[](length)` before the loop and subtract the per-element fee rather than the running total to avoid undercharging and keep each array write in-bounds.
  