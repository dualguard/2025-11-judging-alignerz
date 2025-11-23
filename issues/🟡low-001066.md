# [001066] Self-Merge Causes Unintended NFT Burn, Leading to User-Fund Loss
  
  ### Summary

`mergeTVS()` does not check whether the sourceNftId and destinationNftId are the same.
If a user accidentally passes the same NFT ID for both values, the function treats the NFT as both the source and the target. This causes the source NFT to be burned, which in this case is the same NFT, leading to a complete and irreversible loss of all vesting flows.

### Root Cause

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002

`mergeTVS()` lacks a required validation:
```solidity
require(sourceNftId != destinationNftId, "Cannot merge into itself");
```
Without this check, a self-merge proceeds normally, and the logic that burns the source NFT ends up burning the destination NFT as well since both IDs are identical.

### Internal Pre-conditions

1. User must call mergeTVS() with:
   - sourceNftId == destinationNftId
2. Function lacks a validation preventing self-merge.

### External Pre-conditions

None, the issue arises entirely due to improper input during normal protocol usage.

### Attack Path

1. A user mistakenly calls:
```solidity
mergeTVS(projectId, nftId, [projectId], [nftId]);
```
2. The function proceeds with merging since no validation prevents it.
3. After assigning flows, the contract burns the source NFT.
4. Because `source == destination`, the user’s only NFT is burned.
5. All vesting flows tied to the NFT are permanently lost.

### Impact

The user permanently loses their TVS NFT and all claimable vesting tokens because the NFT is burned accidentally.

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
        vesting.setTreasury(address(1));
        
        uint256 tvsAllocationPerAddress=_totalTVSAllocation/KOLS.length;
        uint256 totalTVSAllocation;
        uint256[] memory TVSamounts=new uint256[](KOLS.length);
        for(uint256 i=0; i<KOLS.length;i++){
            TVSamounts[i]=tvsAllocationPerAddress;
            totalTVSAllocation+=tvsAllocationPerAddress;
        }
        vesting.setTVSAllocation(0, totalTVSAllocation, vestingPeriod, KOLS, TVSamounts);
        vm.stopPrank();
    }
    
    function test_SOURCE_AND_DESTINATION_SAME_NFT_WILL_EFFECTIVELY_BURN_THE_NFT()public{
        address[] memory KOLS=new address[](3);
        KOLS[0] = USER1;
        KOLS[1] = USER2;
        KOLS[2] = USER3;
        uint256 totalTVSAllocation = 9000e18;
        uint256 vestingPeriod=100;
        uint256 rewardProjectId=0;
        createProjectsAndSetAllocation(vestingPeriod,totalTVSAllocation,KOLS);

        hoax(USER1);
        vesting.claimRewardTVS(rewardProjectId);

        uint256 projectId=0;
        uint256 mergedNftId=1;
        uint256[] memory projectIds=new uint256[](1);
        uint256[] memory nftIds=new uint256[](1);
        projectIds[0]=projectId;
        nftIds[0]=mergedNftId;
        address owner=nft.ownerOf(mergedNftId);
        assertEq(owner,USER1);
        hoax(USER1);
        vesting.mergeTVS(projectId, mergedNftId, projectIds, nftIds);

        vm.expectRevert(abi.encodeWithSignature("OwnerQueryForNonexistentToken()"));
        nft.ownerOf(mergedNftId);
    }


}
```

1. Add the above test in `test/vesting/TestAlignerzVesting.t.sol`
2. Run the following command in terminal.
```bash
forge test --mt test_SOURCE_AND_DESTINATION_SAME_NFT_WILL_EFFECTIVELY_BURN_THE_NFT -vv
```
3. You will see the output something simillar to below
```bash
Ran 1 test for test/vesting/TestAlignerzVesting.t.sol:TestAlignerzVesting
[PASS] test_SOURCE_AND_DESTINATION_SAME_NFT_WILL_EFFECTIVELY_BURN_THE_NFT() (gas: 1174292)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.25ms (998.79µs CPU time)
```

### Mitigation

```diff
contract AlignerzVesting{
    .
    .
    .
    .
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
+           if(mergedNftId==nftIds[i]):
+               continue
            feeAmount += _merge(mergedTVS, projectIds[i], nftIds[i], token);
        }
        token.safeTransfer(treasury, feeAmount);
        emit TVSsMerged(projectId, isBiddingProject, nftIds, mergedNftId, mergedTVS.amounts, mergedTVS.vestingPeriods, mergedTVS.vestingStartTimes, mergedTVS.claimedSeconds, mergedTVS.claimedFlows);
        return mergedNftId;
    }
    .
    .
    .
}
```
  