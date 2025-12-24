# [001071] Out-of-Sync allocationOf Mapping Enables Dividend Over-Claiming & Causes Inconsistent TVS State
  
  ### Summary

The root cause is that the `allocationOf` mapping—intended to serve as the authoritative source for a TVS allocation—is not kept in sync when users claim tokens, merge TVSs, or split TVSs.
Because many parts of the system (most importantly the Dividend Distributor) depend on `allocationOf[nftId]` to compute unclaimed token amounts, this stale data allows:

1. **Over-claiming of dividends**
    A user can fully claim all their project tokens right before vesting ends (thus keeping the NFT alive), but because `allocationOf` is never updated, the dividend module still believes the user has the full original allocation.
    The user can then claim dividends as if they still held unclaimed tokens -> they receive dividends for tokens they no longer hold.
2. **User loss during merges and splits**
    Since merges/splits update the RewardProject.allocations state, but not the external-facing `allocationOf`, UI and downstream contracts will read incorrect outdated data, causing inaccurate calculations and user confusion or loss.

### Root Cause

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L114

In the above line, `allocationOf[nftId]` is a public mapping intended to mirror the allocation stored inside either:
- `biddingProjects[projectId].allocations[nftId]`
    Or
- `rewardProjects[projectId].allocations[nftId]`

However:
When users claim their TVS tokens, only the biddingProjects/rewardProjects allocation is updated. `allocationOf[nftId]` remains outdated.
When users merge `TVSs`, the merged allocation is not reflected in `allocationOf`.
When users split TVSs, `allocationOf` is written inconsistently.

Because the dividend logic and other external reads use `allocationOf`, the system effectively relies on stale, incorrect data.

### Internal Pre-conditions

1. A user must have a valid TVS NFT with vesting amounts.
2. The user must be able to call `claim` or `merge` or `split` successfully.
3. After these actions, the allocation stored in `rewardProjects.allocations` or `biddingProjects.allocations` must diverge from `allocationOf`.

### External Pre-conditions

1. A project announces dividends and a Dividend Distributor is deployed.
2. `A26ZDividendDistributor` calls `getUnclaimedAmounts()` which reads `allocationOf[nftId]`.

### Attack Path

1. User claims all tokens just before their vesting completes.Their NFT is not burned.
2. Internal allocation in biddingProjects/rewardProjects is updated to reflect claimedSeconds, but `allocationOf[nftId]` still shows the full original amounts.
3. Dividends are initialized for the project.
4. Dividend logic reads stale `allocationOf[nftId]`.
5. User claims dividends for **both claimed + unclaimed tokens**, over-claiming their share.

### Impact

- Users can over-claim dividends using outdated allocations stored in `allocationOf`. They receive dividends for tokens they no longer have.
- Users will also lose tokens during merges due to incorrect updates.

### PoC

**Note :** Before Running this POC make sure that your `FeesManager::calculateFeeAndNewAmountForOneTVS` and `AlignerzVesting::_computeSplitArrays` looks like below. Because there already exists different issues. Those should be fixed inorder to run this test. 

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
            unchecked {
                i++;
            }
        }
    }
```

The below will be the actual POC

```solidity
//SPDX-License-Identifier: MIT
pragma solidity =0.8.29;


import {Test,console} from "forge-std/Test.sol";
import {ERC20,IERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Aligners26} from "src/contracts/token/Aligners26.sol";
import {AlignerzVesting} from "src/contracts/vesting/AlignerzVesting.sol";
import {AlignerzNFT} from "src/contracts/nft/AlignerzNFT.sol";
import {A26ZDividendDistributor} from "src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {MockUSD} from "src/MockUSD.sol";
import {IAlignerzVesting} from "src/interfaces/IAlignerzVesting.sol";

contract AlignerzVestingForTest is AlignerzVesting{
    function allocationof(uint256 nftId)public view returns(Allocation memory _allocation){
        _allocation =allocationOf[nftId];
    }

    function getRewardProjectsInfo(uint256 projectId)public view returns (address[] memory kolTVSAddresses, address[] memory kolStablecoinAddresses){
        kolTVSAddresses=rewardProjects[projectId].kolTVSAddresses;
        kolStablecoinAddresses=rewardProjects[projectId].kolStablecoinAddresses;
    }

    function getKOLIndexes(uint256 projectId, address KOL,bool isTvs)public returns(uint256){
        if(isTvs){
            return rewardProjects[projectId].kolTVSIndexOf[KOL];
        }
        return rewardProjects[projectId].kolStablecoinIndexOf[KOL];
    }

    function getTotalTokens(uint256 nftId)public view returns(uint256 amount){
        uint256 len =allocationOf[nftId].amounts.length;
        for(uint256 i=0;i<len;i++){
            amount+=allocationOf[nftId].amounts[i];
        }
    } 

    function getTotalTokens(uint256 projectId,uint256 nftId,bool isBiddingProject)public view returns(uint256 amount){
        Allocation memory alloc;
        if(isBiddingProject){
            alloc=biddingProjects[projectId].allocations[nftId];
        }else{
            alloc=rewardProjects[projectId].allocations[nftId];
        }
        uint256 len=alloc.amounts.length;
        for(uint256 i=0;i<len;i++){
            amount+=alloc.amounts[i];
        }
    } 
}
contract TestAlignerzVesting is Test {

    MockUSD public USD;
    Aligners26 public project1;
    Aligners26 public project2;
    AlignerzVestingForTest public vesting;
    AlignerzNFT public nft;
    A26ZDividendDistributor public dividendDistributorProject1;
    mapping(address=>string) addressToKOL;
    address OWNER=makeAddr("OWNER");
    address USER1=makeAddr("USER1");
    address USER2=makeAddr("USER2");
    address USER3=makeAddr("USER3");
    address USER4=makeAddr("USER4");
    address USER5=makeAddr("USER5");
    address USER6=makeAddr("USER6");
    function setUp()public{
        addressToKOL[USER1]="USER1";
        addressToKOL[USER2]="USER2";
        addressToKOL[USER3]="USER3";

        startHoax(OWNER);
        USD=new MockUSD();
        project1=new Aligners26("PROJECT-1","PROJECT-1");
        project2=new Aligners26("PROJECT-2","PROJECT-2");
        vesting = new AlignerzVestingForTest();
        nft=new AlignerzNFT("Aligner-NFT","NFT","NO-URI");
        vesting.initialize(address(nft));
        nft.addMinter(address(vesting));
        vm.stopPrank();
    }

    //@note Iam taking reward projects for making the tests simpler
    function createProjectsAndSetAllocation(uint256 vestingPeriod,uint256 _totalTVSAllocation,address[] memory KOLS)internal{
        startHoax(OWNER);
        project1.approve(address(vesting), type(uint256).max);
        uint256 startTime=block.timestamp+10;
        uint256 claimWindow=block.timestamp+110;
        vesting.launchRewardProject(address(project1), address(USD), startTime, claimWindow);
        
        
        uint256 tvsAllocationPerAddress=_totalTVSAllocation/KOLS.length;
        uint256 totalTVSAllocation;
        uint256[] memory TVSamounts=new uint256[](KOLS.length);
        for(uint8 i=0; i<KOLS.length;i++){
            TVSamounts[i]=tvsAllocationPerAddress;
            totalTVSAllocation+=tvsAllocationPerAddress;
        }
        vesting.setTVSAllocation(0, totalTVSAllocation, vestingPeriod, KOLS, TVSamounts);
        vm.stopPrank();
    }
    
    function test_allocationOf_is_Inconsistent()public{
        address[] memory KOLS=new address[](3);
        KOLS[0] = USER1;
        KOLS[1] = USER2;
        KOLS[2] = USER3;
        uint256 totalTVSAllocation = 9000e18;
        uint256 vestingPeriod=100;
        createProjectsAndSetAllocation(vestingPeriod,totalTVSAllocation,KOLS);
        uint256 rewardProjectId=0;
        uint256 startNftId=1;
        uint256[] memory mergeProjectIds=new uint256[](2);
        uint256[] memory mergeNftIds=new uint256[](2);
        uint256 currentNftId=startNftId;
        for(uint8 i=0;i<KOLS.length;i++){
            hoax(KOLS[i]);
            vesting.claimRewardTVS(rewardProjectId);
            if(KOLS[i]!=USER1){
                hoax(KOLS[i]);
                nft.transferFrom(KOLS[i], USER1, currentNftId);
                mergeProjectIds[i-1]=rewardProjectId;
                mergeNftIds[i-1]=currentNftId;

            }
            currentNftId++;
        }

        // Now user1 is the owner of three tokens.
        hoax(OWNER);
        vesting.setTreasury(address(1)); // Setting treasury, else: during fee transfer it will revert due to zero address
        uint256 totalTokensBeforeMerge=vesting.getTotalTokens(startNftId);
        assertEq(vesting.getTotalTokens(startNftId), vesting.getTotalTokens(rewardProjectId,startNftId,false));
        // Before merge the allocationOf[nftId] and rewardProjects[projectId].allocations[nftId] will be same
        // Once merge is done only rewardProjects[projectId].allocations[nftId]  is updated

        hoax(USER1);
        vesting.mergeTVS(rewardProjectId, startNftId, mergeProjectIds, mergeNftIds);
        console.log(vesting.getTotalTokens(startNftId));
        console.log(vesting.getTotalTokens(rewardProjectId,startNftId,false));
        
        // @note vesting.getTotalTokens(startNftId) will get total tokens based on allocationOf
        // vesting.getTotalTokens(rewardProjectId,startNftId,false) will get total tokens based on 
        // rewardProjects[projectId].allocations[nftId]

        assertEq(vesting.getTotalTokens(rewardProjectId,startNftId,false), totalTVSAllocation);
        assertEq(vesting.getTotalTokens(startNftId), totalTokensBeforeMerge);

        // The sameway it can be exploited by an malicious user in claimTokens function. 
    }


}
```

1. Add the above test in `test/vesting/TestAlignerzVesting.t.sol`
2. Run the following command in terminal.
```bash
forge test --mt test_allocationOf_is_Inconsistent -vv
```
3. You will see the output something simillar to below
```bash
Ran 1 test for test/vesting/TestAlignerzVesting.t.sol:TestAlignerzVesting
[PASS] test_allocationOf_is_Inconsistent() (gas: 2323653)
Logs:
  3000000000000000000000
  9000000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.56ms (2.05ms CPU time)
```

### Mitigation

```diff
contract AlignerzVesting{
    .
    .
    .
     function claimTokens(uint256 projectId, uint256 nftId) external {
        address nftOwner = nftContract.extOwnerOf(nftId);
        require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());
        bool isBiddingProject = NFTBelongsToBiddingProject[nftId];
        (Allocation storage allocation, IERC20 token) = isBiddingProject ? 
        (biddingProjects[projectId].allocations[nftId], biddingProjects[projectId].token) : 
        (rewardProjects[projectId].allocations[nftId], rewardProjects[projectId].token);
        uint256 nbOfFlows = allocation.vestingPeriods.length;
        uint256 claimableAmounts;
        uint256[] memory amountsClaimed = new uint256[](nbOfFlows);
        uint256[] memory allClaimableSeconds = new uint256[](nbOfFlows);
        uint256 flowsClaimed;
        for (uint256 i; i < nbOfFlows; i++) {
            if (allocation.claimedFlows[i]) {
                flowsClaimed++;
                continue;
            }
            (uint256 claimableAmount, uint256 claimableSeconds) = getClaimableAmountAndSeconds(allocation, i);

            allocation.claimedSeconds[i] += claimableSeconds;
            if (allocation.claimedSeconds[i] >= allocation.vestingPeriods[i]) {
                flowsClaimed++;
                allocation.claimedFlows[i] = true;
            }
            allClaimableSeconds[i] = claimableSeconds;
            amountsClaimed[i] = claimableAmount;
            claimableAmounts += claimableAmount;
        }
        if (flowsClaimed == nbOfFlows) {
            nftContract.burn(nftId);
            allocation.isClaimed = true;
        }
+       allocationOf[nftId]=allocation;
        token.safeTransfer(msg.sender, claimableAmounts);
        emit TokensClaimed(projectId, isBiddingProject, allocation.assignedPoolId, allocation.isClaimed, nftId, allClaimableSeconds, block.timestamp, msg.sender, amountsClaimed);
    }

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
        }
        token.safeTransfer(treasury, feeAmount);
+       allocationOf[mergedNftId]=mergedTVS;
        emit TVSsMerged(projectId, isBiddingProject, nftIds, mergedNftId, mergedTVS.amounts, mergedTVS.vestingPeriods, mergedTVS.vestingStartTimes, mergedTVS.claimedSeconds, mergedTVS.claimedFlows);
        return mergedNftId;
    }

}
```
  