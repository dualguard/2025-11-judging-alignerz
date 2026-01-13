# [001076] KOL Allocation Functions Lack “Single-Use” Guard, Allowing Double Allocation per Project
  
  ### Summary

Both `setTVSAllocation()` and `setStablecoinAllocation()` are designed to initialize KOL allocations for a project. These functions should logically be called only once per project, because they define the full reward setup for that project.
However, the implementation does not include any check to ensure they are executed only once.
Because of this, the owner can unintentionally call these functions multiple times for the same `rewardProjectId`. Doing so appends new KOL entries, overwrites mappings, and overwrites the vestingTime. 

There will be two impacts because of this. I have detailed the impact clearly in Attack Path, POC and Impact. 

### Root Cause

In both allocation functions:

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L458

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L484

there is no guard such as:
```solidity
require(!rewardProject.allocationSet, "Allocation already initialized");
```



### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

Attack Path -1
1. Owner calls setTVSAllocation(projectId, [...]) with KOLs A, B, C → stored indices become 0,1,2.
2. Owner calls setTVSAllocation(projectId, [...]) again with KOLs D, E, F → they are appended at actual indices 3,4,5, but stored indices are incorrectly set to 0,1,2.
3. KOL D calls claimRewardTVS() → the contract looks up kolTVSIndexOf[D] = 0 even though D is actually at index 3.
4. The swap-and-pop logic removes/replaces index 0 instead of index 3.
5. The KOL originally at index 0 (KOL A) is overwritten and permanently removed from the reward list.
6. The KOL A wont call `claimRewardTVS` during `claimPeriod`.
7. Once the `claimPeriod` is passed owner calls `distributeRemainingRewardTVS()` → KOL A will never receive their TVS reward. But D again removes a TVS nft (worth 0) because their address is not removed from the array.

Attack Path -2

1. Owner calls `setTVSAllocation()` with vestingPeriod = 300 for KOLs A, B, C.
2. Before A/B/C claim, owner calls `setTVSAllocation()` again for KOL D with vestingPeriod = 30.
3. This overwrites `rewardProject.vestingPeriod = 30`.
4. A/B/C eventually call `claimRewardTVS()`.
5. Their NFTs are minted with vestingPeriod = 30 instead of 300.
6. A/B/C’s vesting is shortened by 90%, unlocking tokens much earlier than intended.

### Impact

Impact -1
The affected KOLs cannot receive their allocated TVS rewards, resulting in a loss of funds owed to them.

Impact -2
This flaw breaks the core IWO vesting model because earlier KOLs do not keep their original vesting periods. Their vesting duration gets overwritten whenever the owner allocates a new batch. As a result, earlier KOLs may unlock far earlier or far later than intended, making the vesting schedule inconsistent and invalid across batches.


### PoC

POC-1
```solidity
//SPDX-License-Identifier: MIT
pragma solidity 0.8.29;


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
    function createProjectsAndSetAllocation()internal{
        startHoax(OWNER);
        project1.approve(address(vesting), type(uint256).max);
        project2.approve(address(vesting), type(uint256).max);
        uint256 startTime=block.timestamp+10;
        uint256 claimWindow=block.timestamp+110;
        vesting.launchRewardProject(address(project1), address(USD), startTime, claimWindow);
        
        address[] memory KOLS=new address[](3);
        KOLS[0] = USER1;
        KOLS[1] = USER2;
        KOLS[2] = USER3;
        uint256 _totalTVSAllocation=9000e18;
        uint256 tvsAllocationPerAddress=_totalTVSAllocation/KOLS.length;
        uint256 totalTVSAllocation;
        uint256[] memory TVSamounts=new uint256[](KOLS.length);
        for(uint8 i=0; i<KOLS.length;i++){
            TVSamounts[i]=tvsAllocationPerAddress;
            totalTVSAllocation+=tvsAllocationPerAddress;
        }
        vesting.setTVSAllocation(0, totalTVSAllocation, 100, KOLS, TVSamounts);
        vm.stopPrank();
    }

    function allocationTwo()public{
        
        address[] memory KOLS=new address[](3);
        KOLS[0] = USER4;
        KOLS[1] = USER5;
        KOLS[2] = USER6;
        
        startHoax(OWNER);
        uint256 _totalTVSAllocation=9000e18;
        uint256 tvsAllocationPerAddress=_totalTVSAllocation/KOLS.length;
        uint256 totalTVSAllocation;
        uint256[] memory TVSamounts=new uint256[](KOLS.length);
        for(uint8 i=0; i<KOLS.length;i++){
            TVSamounts[i]=tvsAllocationPerAddress;
            totalTVSAllocation+=tvsAllocationPerAddress;
        }
        vesting.setTVSAllocation(0, totalTVSAllocation, 100, KOLS, TVSamounts);
        vm.stopPrank();
    }   
    
    function test_claimRewardTVS_Removes_Incorrect_Indexes()public{
        createProjectsAndSetAllocation();
        allocationTwo();
        assert(vesting.getKOLIndexes(0, USER1, true)==vesting.getKOLIndexes(0, USER4, true));
        assert(vesting.getKOLIndexes(1, USER2, true)==vesting.getKOLIndexes(1, USER5, true));
        assert(vesting.getKOLIndexes(2, USER3, true)==vesting.getKOLIndexes(2, USER6, true));

        hoax(USER4);
        //Since USER4 is being claimed, once the amounts are calculated it will store Last index 
        //address in kolTVSAddresses at the current USER4 index. and it also stores kolTVSIndexOf 
        // USER4 with last index and stores kolTVSIndexOf last user with USER4 current index.
        /*

        ///////////// BEFORE UPDATION /////////////////////

        let USER_4_CURR_INDEX_IN_kolTVSAddresses= 3
        let LAST_USER_CURR_INDEX_IN_kolTVSAddresses=5
        let USER_4_CURR_kolTVSIndexOf=0    *Refer to above asserts*
        let LAST_USER_kolTVSIndexOf=2   *Refer to above asserts*
        let kolTVSAddresses_Array_Length=6

        ///////////// AFTER UPDATION /////////////////////
        let USER_4_UPDATED_kolTVSIndexOf= kolTVSAddresses_Array_Length -1=5
        let LAST_USER_UPDATED_kolTVSIndexOf= USER_4_CURR_kolTVSIndexOf =0
        let LAST_USER_UPDATED_INDEX_IN_kolTVSAddresses=USER_4_CURR_kolTVSIndexOf=0

        When USER4 claims their tvs, the last(USER6) position  must be stored in previous 
        USER4 position and last element should be popped of but since USER_4_CURR_kolTVSIndexOf 
        is reffering to wrong_index (0) the last position will be stored in USER1 position. So 
        USER1 info is technically removed from kolTVSIndexOf and kolTVSAddresses arrays

        */
        vesting.claimRewardTVS(0);
        //Because of the bug when owner calls distributeRemainingRewardTVS once the claimPeriod
        // is end USER1 wont recieve any TVS because USER1 address is removed from kolTVSAddresses
        // array and distributeRemainingRewardTVS will iterate through each address in kolTVSAddresses
        vm.warp(block.timestamp+150);
        hoax(OWNER);
        vesting.distributeRemainingRewardTVS(0);
        assert(nft.balanceOf(USER1)==0);
        assert(nft.balanceOf(USER2)==1);
        assert(nft.balanceOf(USER3)==1);
        assert(nft.balanceOf(USER4)==2);
        assert(nft.balanceOf(USER5)==1);
        assert(nft.balanceOf(USER6)==1);

        address[] memory kol=new address[](1);
        kol[0]=USER1;
        
        // Since distributeRewardTVS iterates trhough rewardProject.kolTVSAddresses.length and 
        //once distributeRemainingRewardTVS is called it will make kolTVSAddresses array zero
        // length. So There is no way user1 can claim there tokens
        vesting.distributeRewardTVS(0, kol);
    }
}
```

1. Add the above test code in `test/vesting/TestAlignerzVesting.t.sol`.
2. Run the following in terminal
```bash
forge test --mt test_claimRewardTVS_Removes_Incorrect_Indexes
```
3. You will get output that looks like the below one
```bash
Ran 1 test for test/vesting/TestAlignerzVesting.t.sol:TestAlignerzVesting
[PASS] test_claimRewardTVS_Removes_Incorrect_Indexes() (gas: 3519726)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.32ms (2.76ms CPU time)
```



POC-2
```solidity
//SPDX-License-Identifier: MIT
pragma solidity 0.8.29;


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
        project2.approve(address(vesting), type(uint256).max);
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

    function allocationTwo(uint256 vestingPeriod)public{
        
        address[] memory KOLS=new address[](3);
        KOLS[0] = USER4;
        KOLS[1] = USER5;
        KOLS[2] = USER6;
        
        startHoax(OWNER);
        uint256 _totalTVSAllocation=9000e18;
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

    function test_InConsistent_Vesting_Period_Cause_Early_Claims()public{
        address[] memory KOLS=new address[](3);
        KOLS[0] = USER1;
        KOLS[1] = USER2;
        KOLS[2] = USER3;
        uint256 totalTVSAllocation = 9000e18;
        uint256 totalTVSAllocationPerUserInAllocation1=totalTVSAllocation/KOLS.length;
        uint256 vestingPeriod1 =300 days;
        createProjectsAndSetAllocation(vestingPeriod1,totalTVSAllocation,KOLS);
        uint256 vestingPeriod2 =30 days;
        allocationTwo(vestingPeriod2);

        uint256 rewardProjectId=0;
        
        uint256 user0NftId=nft.totalSupply()+1;
        startHoax(USER1);
        vesting.claimRewardTVS(rewardProjectId);
    

        vm.warp(block.timestamp + 30.1 days);
        uint256 balanceBeforeClaim= project1.balanceOf(USER1);

        
        vesting.claimTokens(rewardProjectId, user0NftId);
        uint256 balanceAfterClaim= project1.balanceOf(USER1);

        console.log(balanceAfterClaim-balanceBeforeClaim);
        assertEq(balanceAfterClaim, totalTVSAllocationPerUserInAllocation1);

        vm.stopPrank();
    }
}
```

1. Add the above test to test/vesting/TestAlignerzVesting.t.sol
2. Run the following command in terminal
```bash
forge test --mt test_InConsistent_Vesting_Period_Cause_Early_Withdraws -vv
```
3. You will see output something like below
```bash
[PASS] test_InConsistent_Vesting_Period_Cause_Early_Withdraws() (gas: 1386197)
Logs:
  3000000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 9.77ms (2.39ms CPU time)
```

### Mitigation

```diff
contract AlignerzVesting{
    .
    .
    .
    .

   struct RewardProject {
        IERC20 token; // The TVS token
        IERC20 stablecoin; // The stablecoin
        uint256 vestingPeriod; // The vesting period is the same for all KOLs
        mapping(uint256 => Allocation) allocations; // Mapping of NFT token ID to allocations
        mapping(address => uint256) kolTVSRewards; // Mapping to track the allocated TVS rewards for each KOL
        mapping(address => uint256) kolStablecoinRewards; // Mapping to track the allocated stablecoin rewards for each KOL
        mapping(address => uint256) kolTVSIndexOf; // Mapping to track the KOL address index position inside kolTVSAddresses
        mapping(address => uint256) kolStablecoinIndexOf; // Mapping to track the KOL address index position inside kolStablecoinAddresses
        address[] kolTVSAddresses; // array of kol addresses that are yet to claim TVS allocation
        address[] kolStablecoinAddresses; // array of kol addresses that are yet to claim their stablecoin allocation
        uint256 startTime; // startTime of the vesting periods
        uint256 claimDeadline; // deadline after which it's impossible for users to claim TVS or refund
+       bool TVSAllocated;
+       bool StableCoinAllocated;
   }
    .
    .
    .
    .
    /// @notice Sets KOLs TVS allocations
    /// @param rewardProjectId Id of the rewardProject
    /// @param totalTVSAllocation total amount to be allocated in TVSs to KOLs
    /// @param vestingPeriod duration the vesting periods
    /// @param kolTVS addresses of the KOLs who chose to be rewarded in TVS
    /// @param TVSamounts token amounts allocated for the KOLs who chose to be rewarded in TVS
    function setTVSAllocation(uint256 rewardProjectId, uint256 totalTVSAllocation, uint256 vestingPeriod, address[] calldata kolTVS, uint256[] calldata TVSamounts) external onlyOwner {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
+       require(!rewardProject.TVSAllocated,"TVS Already Allocated");
+       rewardProject.TVSAllocated=true;
        rewardProject.vestingPeriod = vestingPeriod;
        uint256 length = kolTVS.length;
        require(length == TVSamounts.length, Array_Lengths_Must_Match());
        uint256 totalAmount;
        for (uint256 i = 0; i < length; i++) {
            address kol = kolTVS[i];
            rewardProject.kolTVSAddresses.push(kol);
            uint256 amount = TVSamounts[i];
            rewardProject.kolTVSRewards[kol] = amount;
            rewardProject.kolTVSIndexOf[kol] = i;
            totalAmount += amount;
            emit TVSAllocated(rewardProjectId, kol, amount, vestingPeriod);
        }
        require(
            totalTVSAllocation == totalAmount, Amounts_Do_Not_Add_Up_To_Total_Allocation()
        );
        rewardProject.token.safeTransferFrom(msg.sender, address(this), totalTVSAllocation);
    }
    .
    .
    .
    .
    /// @notice Sets KOLs Stablecoin allocations
    /// @param rewardProjectId Id of the rewardProject
    /// @param totalStablecoinAllocation total amount to be allocated in stablecoin to KOLs
    /// @param kolStablecoin addresses of the KOLs who chose to be rewarded in stablecoin
    /// @param stablecoinAmounts stablecoin amounts allocated for the KOLs who chose to be rewarded in stablecoin
    function setStablecoinAllocation(uint256 rewardProjectId, uint256 totalStablecoinAllocation, address[] calldata kolStablecoin, uint256[] calldata stablecoinAmounts) external onlyOwner {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
+       require(!rewardProject.StableCoinAllocated,"Stable Coin Already Allocated");
+       rewardProject.StableCoinAllocated=true;
        uint256 length = kolStablecoin.length;
        require(length == stablecoinAmounts.length, Array_Lengths_Must_Match());
        uint256 totalAmount;
        for (uint256 i = 0; i < length; i++) {
            address kol = kolStablecoin[i];
            rewardProject.kolStablecoinAddresses.push(kol);
            uint256 amount = stablecoinAmounts[i];
            rewardProject.kolStablecoinRewards[kol] = amount;
            rewardProject.kolStablecoinIndexOf[kol] = i;
            totalAmount += amount;
            emit StablecoinAllocated(rewardProjectId, kol, amount);
        }
        require(
            totalStablecoinAllocation == totalAmount, Amounts_Do_Not_Add_Up_To_Total_Allocation()
        );
        rewardProject.stablecoin.safeTransferFrom(msg.sender, address(this), totalStablecoinAllocation);
    }
}
```
  