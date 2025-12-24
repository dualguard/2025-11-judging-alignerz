# [000748] Merging an NFT into itself burns the only TVS and causes loss of funds
  
  
### Summary

Allowing `mergeTVS` to use the same NFT as both the merge target and a source will cause loss of funds for the user as `_merge` burns the target NFT, leaving its allocation stranded and `claimTokens` permanently reverting.

### Root Cause

In `_merge`, there is no protection against `nftId == mergedNftId`, so if the user passes the same NFT as both the `mergedNftId` and as one of the `nftIds[]`, the function operates on the same allocation and burns the merged NFT itself:

```solidity
function _merge(Allocation storage mergedTVS, uint256 projectId, uint256 nftId, IERC20 token) internal returns (uint256 feeAmount) {
    require(msg.sender == nftContract.extOwnerOf(nftId), Caller_Should_Own_The_NFT());
    
    bool isBiddingProjectTVSToMerge = NFTBelongsToBiddingProject[nftId];
    (Allocation storage TVSToMerge, IERC20 tokenToMerge) = isBiddingProjectTVSToMerge ?
        (biddingProjects[projectId].allocations[nftId], biddingProjects[projectId].token) :
        (rewardProjects[projectId].allocations[nftId], rewardProjects[projectId].token);
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
>>  nftContract.burn(nftId);
}
```

In `mergeTVS`, `mergedTVS` is an alias to the allocation of `mergedNftId`. If `nftIds[i]` contains the same `mergedNftId`, then `TVSToMerge` inside `_merge` refers to the same allocation, and after pushing duplicate flows, the function calls `nftContract.burn(nftId)` on the merged NFT itself. There is no subsequent re-mint; the mapping `allocationOf[mergedNftId]` still holds data but the NFT is burned, so `claimTokens` relying on `extOwnerOf(nftId)` will revert forever.

### Internal Pre-conditions

1. A user must own a TVS NFT `nftId1` with a non-zero allocation in either a reward or bidding project.  
2. The user must be able to call `mergeTVS(projectId, mergedNftId, projectIds, nftIds)` where `mergedNftId == nftId1` and `projectIds[0] = projectId`, `nftIds[0] = nftId1`.

### External Pre-conditions

1. No other protocol condition is required; only that the user can send a transaction with arbitrary `mergedNftId` and `nftIds` they own.

### Attack Path

1. The user obtains a TVS NFT `nftId1` via normal flows (e.g. `claimRewardTVS` or `claimNFT`).  
2. The user mistakenly or intentionally calls `mergeTVS(projectId, nftId1, [projectId], [nftId1])`, using the same NFT as both target and source.  
3. Inside `mergeTVS`, `mergedTVS` is `allocations[nftId1]`.  
4. `_merge` is called with that same `nftId1`; `TVSToMerge` is also `allocations[nftId1]`, and after pushing fee-reduced duplicates of its own flows, `_merge` executes `nftContract.burn(nftId1)`.  
5. After `mergeTVS` returns, the NFT `nftId1` is burned; attempts to call `claimTokens(projectId, nftId1)` will revert on `nftContract.extOwnerOf(nftId1)`, so the user can no longer claim any of their vested tokens.

### Impact

A user can accidentally brick their own TVS position and lose access to all vested tokens associated with that NFT by passing incorrect parameters to `mergeTVS`; the protocol does not prevent nor warn about merging an NFT into itself leading to permanent loss of claimability for that TVS.

### PoC

This is demonstrated by the passing Foundry test in `test/MergeSelfBurnBug.t.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

contract MergeSelfBurnBugTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;

    address public owner;

    function setUp() public {
        owner = address(this);

        usdt = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        AlignerzVesting implementation = new AlignerzVesting();

        bytes memory initData = abi.encodeCall(AlignerzVesting.initialize, (address(nft)));
        ERC1967Proxy proxy = new ERC1967Proxy(address(implementation), initData);
        vesting = AlignerzVesting(payable(address(proxy)));

        nft.addMinter(address(proxy));
        vesting.setTreasury(address(1));

        token.approve(address(vesting), type(uint256).max);
    }

    function test_merge_with_self_burns_tvs_and_blocks_claim() public {
        address kol1 = makeAddr("kol1");

        uint256 rewardProjectId = vesting.rewardProjectCount();
        uint256 startTime = block.timestamp;
        uint256 claimWindow = 30 days;

        vesting.launchRewardProject(address(token), address(usdt), startTime, claimWindow);

        address[] memory kolTVS = new address[](1);
        kolTVS[0] = kol1;

        uint256[] memory TVSamounts = new uint256[](1);
        TVSamounts[0] = 100 ether;

        uint256 totalTVSAllocation = TVSamounts[0];
        uint256 vestingPeriod = 30 days;

        token.transfer(address(vesting), totalTVSAllocation);
        vesting.setTVSAllocation(
            rewardProjectId,
            totalTVSAllocation,
            vestingPeriod,
            kolTVS,
            TVSamounts
        );

        vm.prank(kol1);
        vesting.claimRewardTVS(rewardProjectId);
        uint256 nftId1 = nft.getTotalMinted();

        uint256[] memory projectIds = new uint256[](1);
        projectIds[0] = rewardProjectId;
        uint256[] memory nftIds = new uint256[](1);
        nftIds[0] = nftId1;

        vm.prank(kol1);
        vesting.mergeTVS(rewardProjectId, nftId1, projectIds, nftIds);

        vm.prank(kol1);
        vm.expectRevert();
        vesting.claimTokens(rewardProjectId, nftId1);
    }
}
```

Run this test with:

```bash
forge test --mt test_merge_with_self_burns_tvs_and_blocks_claim -vvvv
```

Result:
```bash
Ran 1 test for test/MergeSelfBurnBug.t.sol:MergeSelfBurnBugTest
[PASS] test_merge_with_self_burns_tvs_and_blocks_claim() (gas: 957461)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.55ms (469.79µs CPU time)
```

### Mitigation

Add an explicit check in `mergeTVS` to forbid merging the target NFT into itself, and optionally guard against other self-referential patterns:

```solidity
function mergeTVS(
    uint256 projectId,
    uint256 mergedNftId,
    uint256[] calldata projectIds,
    uint256[] calldata nftIds
) external returns (uint256) {
    ...
    uint256 nbOfNFTs = nftIds.length;
    require(nbOfNFTs > 0, Not_Enough_TVS_To_Merge());
    require(nbOfNFTs == projectIds.length, Array_Lengths_Must_Match());

    for (uint256 i; i < nbOfNFTs; i++) {
        require(nftIds[i] != mergedNftId, "Cannot merge NFT into itself");
        feeAmount += _merge(mergedTVS, projectIds[i], nftIds[i], token);
    }
    ...
}
```


  