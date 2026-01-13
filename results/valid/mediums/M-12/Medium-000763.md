# [000763] Zero amount reward allocations create junk TVSs bricking `claimTokens` when merged
  
  ### Summary

Zero-amount reward allocations combined with the strict `No_Claimable_Tokens` check will cause a permanent inability to claim tokens from merged TVSs as any merged TVS containing a zero-amount flow will always revert when claiming.

### Root Cause

In `AlignerzVesting.sol:979-994` the function `getClaimableAmountAndSeconds` enforces `claimableAmount > 0` for every flow, and in `distributeRemainingRewardTVS` the admin can mint NFTs with `amount = 0`, which later propagate into merges and make the resulting merged TVS unclaimable.

```solidity
    function distributeRemainingRewardTVS(uint256 rewardProjectId) external onlyOwner{ //@audit L better nonreentrant due to minting
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
        uint256 len = rewardProject.kolTVSAddresses.length;
        for (uint256 i = len - 1; rewardProject.kolTVSAddresses.length > 0;) {
            address kol = rewardProject.kolTVSAddresses[i];
>>          uint256 amount = rewardProject.kolTVSRewards[kol]; //@audit amount is never checked to be gt 0
            rewardProject.kolTVSRewards[kol] = 0;
            uint256 nftId = nftContract.mint(kol); 
            rewardProject.allocations[nftId].amounts.push(amount);
            uint256 vestingPeriod = rewardProject.vestingPeriod;
            rewardProject.allocations[nftId].vestingPeriods.push(vestingPeriod);
            rewardProject.allocations[nftId].vestingStartTimes.push(rewardProject.startTime);
            rewardProject.allocations[nftId].claimedSeconds.push(0);
            rewardProject.allocations[nftId].claimedFlows.push(false);
            rewardProject.allocations[nftId].token = rewardProject.token;
            rewardProject.kolTVSAddresses.pop();
            allocationOf[nftId] = rewardProject.allocations[nftId];
            emit RewardTVSDistributed(rewardProjectId, kol, nftId, amount, vestingPeriod);
            unchecked {
                --i;
            }
        }
    }
```

```solidity
    function getClaimableAmountAndSeconds(Allocation memory allocation, uint256 flowIndex) public view returns(uint256 claimableAmount, uint256 claimableSeconds) {
        uint256 secondsPassed;
        uint256 claimedSeconds = allocation.claimedSeconds[flowIndex];
        uint256 vestingPeriod = allocation.vestingPeriods[flowIndex];
        uint256 vestingStartTime = allocation.vestingStartTimes[flowIndex];
        uint256 amount = allocation.amounts[flowIndex];
        if (block.timestamp > vestingPeriod + vestingStartTime) {
            secondsPassed = vestingPeriod;
        } else {
            secondsPassed = block.timestamp - vestingStartTime;
        }
        claimableSeconds = secondsPassed - claimedSeconds;
        claimableAmount = (amount * claimableSeconds) / vestingPeriod;
>>      require(claimableAmount > 0, No_Claimable_Tokens()); //@audit admin created junk nfts will block here if merged!
        return (claimableAmount, claimableSeconds);
    }
```

Because `setTVSAllocation` does not forbid zero-amount allocations, the owner can (even accidentally) configure some KOLs with `amount = 0`. Later, `distributeRemainingRewardTVS` will mint NFTs for those KOLs with a zero-amount flow. When such a zero-amount NFT is merged into a valid TVS via `mergeTVS`, the resulting merged TVS contains at least one zero-amount flow; when `claimTokens` iterates flows and calls `getClaimableAmountAndSeconds`, it eventually hits the zero-amount flow and the `require(claimableAmount > 0, No_Claimable_Tokens())` makes the entire claim revert, permanently blocking the user from claiming any tokens from that merged NFT.

### Internal Pre-conditions

1. The owner must configure a reward project with at least one KOL having a strictly positive TVS allocation and at least one other KOL with a zero TVS allocation (for example via `setTVSAllocation` with `TVSamounts = [100 ether, 0]`).  
2. After the claim window, the owner must call `distributeRemainingRewardTVS(rewardProjectId)`, minting one NFT with positive `amount > 0` and another NFT with `amount = 0` in their `Allocation.amounts` arrays.

### External Pre-conditions

1. No external oracle or price condition is required; the issue arises solely from internal vesting configuration and the passage of time beyond the reward project’s `claimDeadline`.  


### Attack Path

1. The admin launches a reward project and calls `setTVSAllocation` with `kolTVS = [kol1, kol2]` and `TVSamounts = [100 ether, 0]`, so that `rewardProject.kolTVSRewards[kol1] = 100 ether` and `rewardProject.kolTVSRewards[kol2] = 0`.   (this is for easy POC, kol2 could have claimed their amounts and zeroed that out too)
2. After the claim window expires, the admin calls `distributeRemainingRewardTVS(rewardProjectId)`, which mints one NFT for `kol1` with `Allocation.amounts = [100 ether]` and another NFT for `kol2` with `Allocation.amounts = [0]`.  
3. `kol2` transfers their zero-amount NFT to `kol1`, so `kol1` now owns both the positive NFT and the zero-amount NFT.  
4. `kol1` calls `mergeTVS(rewardProjectId, nftIdPositive, [rewardProjectId], [nftIdZero])`, which appends the zero-amount flow to the positive flows in the merged TVS’ `Allocation.amounts` array.  
5. After the vesting period has fully elapsed, `kol1` calls `claimTokens(rewardProjectId, nftIdPositive)` to claim their vested TVS.  
6. Inside `claimTokens`, for the merged TVS, the loop over flows eventually reaches the zero-amount flow; `getClaimableAmountAndSeconds` computes `claimableAmount = 0` for that flow and then hits `require(claimableAmount > 0, No_Claimable_Tokens())`, which reverts the entire transaction, permanently blocking `kol1` from claiming any tokens from that merged TVS.

### Impact

A user holding a valid positive-amount TVS who merges it with a zero-amount “junk” TVS will lose the ability to claim any of their vested tokens from the merged TVS, as every `claimTokens` call on that NFT will revert due to the zero-amount flow, causing a denial of service on withdrawals for that position.

### PoC

Insert below under `test` folder and run with `forge test --mt test_zero_amount_nft_merge_blocks_claim_tokens -vvv`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

contract ZeroAmountMergeJunkBugTest is Test {
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

    function test_zero_amount_nft_merge_blocks_claim_tokens() public {
        address kol1 = makeAddr("kol1");
        address kol2 = makeAddr("kol2");

        uint256 rewardProjectId = vesting.rewardProjectCount();
        uint256 startTime = block.timestamp;
        uint256 claimWindow = 30 days;

        vesting.launchRewardProject(address(token), address(usdt), startTime, claimWindow);

        // kol1 gets a positive TVS allocation, kol2 gets a zero TVS allocation
        address[] memory kolTVS = new address[](2);
        kolTVS[0] = kol1;
        kolTVS[1] = kol2;

        uint256[] memory TVSamounts = new uint256[](2);
        TVSamounts[0] = 100 ether;
        TVSamounts[1] = 0;

        uint256 totalTVSAllocation = TVSamounts[0] + TVSamounts[1];
        uint256 vestingPeriod = 30 days;

        token.transfer(address(vesting), totalTVSAllocation);
        vesting.setTVSAllocation(
            rewardProjectId,
            totalTVSAllocation,
            vestingPeriod,
            kolTVS,
            TVSamounts
        );

        // After the claim window, admin calls distributeRemainingRewardTVS
        vm.warp(startTime + claimWindow + 1);
        vm.prank(owner);
        vesting.distributeRemainingRewardTVS(rewardProjectId);

        // Two NFTs were minted by distributeRemainingRewardTVS in order:
        // first for kol2 (id = totalMinted - 1), then for kol1 (id = totalMinted).
        uint256 totalMinted = nft.getTotalMinted();
        uint256 nftIdKol1 = totalMinted;
        uint256 nftIdKol2 = totalMinted - 1;

        // Sanity check the inferred owners
        assertEq(nft.ownerOf(nftIdKol1), kol1);
        assertEq(nft.ownerOf(nftIdKol2), kol2);

        // Transfer the zero-amount NFT to kol1 so kol1 owns both NFTs and can merge them
        vm.prank(kol2);
        nft.transferFrom(kol2, kol1, nftIdKol2);

        // Merge the zero-amount NFT into the positive NFT
        uint256[] memory projectIds = new uint256[](1);
        projectIds[0] = rewardProjectId;
        uint256[] memory nftIds = new uint256[](1);
        nftIds[0] = nftIdKol2;

        vm.prank(kol1);
        vesting.mergeTVS(rewardProjectId, nftIdKol1, projectIds, nftIds);

        // After vesting ends, trying to claim tokens from the merged NFT reverts,
        // because getClaimableAmountAndSeconds() hits a zero-amount flow and reverts.
        vm.warp(startTime + vestingPeriod + claimWindow + 2);

        vm.prank(kol1);
        vm.expectRevert();
        vesting.claimTokens(rewardProjectId, nftIdKol1);
    }
}
```

Result:

```bash
Ran 1 test for test/ZeroAmountMergeJunkBug.t.sol:ZeroAmountMergeJunkBugTest
[PASS] test_zero_amount_nft_merge_blocks_claim_tokens() (gas: 1542335)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.75ms (651.38µs CPU time)

Ran 1 test suite in 84.05ms (1.75ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
   
```


### Mitigation

A minimal and robust fix is to prevent zero-amount flows from ever being created in the first place and to relax the `No_Claimable_Tokens` check so that zero-amount or fully-vested-but-empty flows do not cause a revert.


  