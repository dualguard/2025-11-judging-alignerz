# [001073] _computeSplitArrays Always Reverts Due to Zero-Length Dynamic Arrays in Returned Allocation
  
  ### Summary

The `_computeSplitArrays` function constructs and returns an Allocation struct. However, because `alloc` is declared as a memory return variable, all dynamic array fields inside the `Allocation` struct (`amounts, vestingPeriods, vestingStartTimes, claimedSeconds, claimedFlows`) are initialized with zero length.

The function then immediately attempts to write into these arrays:
```solidity
alloc.amounts[j] = ...
```
Since all arrays have length 0, any write operation will **revert with array out-of-bounds access**.

This creates a denial-of-service condition in the vesting system, breaking one of Alignerz’s core token-management features.

### Root Cause

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1121

In AlignerzVesting.sol, within _computeSplitArrays:
```solidity
Allocation memory alloc;
```
This initializes:
- alloc.amounts -> empty array (length = 0)
- alloc.vestingPeriods -> empty array
- alloc.vestingStartTimes -> empty array
- alloc.claimedSeconds -> empty array
- alloc.claimedFlows -> empty array

But the function tries to write into them:
```solidity
But the function tries to write into them:
```
This is an out-of-bounds write because the arrays were never allocated with a size using:
```solidity
alloc.amounts = new uint256[](nbOfFlows);
```

### Internal Pre-conditions

1. A TVS containing nbOfFlows > 0 exists.
2. A function call to `splitTVs`, which internally calls _computeSplitArrays.

### External Pre-conditions

None, this bug triggers on every call regardless of state.


### Attack Path

1. A user attempts to split a TVS using splitTVS.
2. `splitTVS` internally calls `_computeSplitArrays(...)`.
3. `_computeSplitArrays` creates an `Allocation memory alloc` with all dynamic arrays sized to 0.
4. The function enters the loop and attempts:
    ```solidity
    alloc.amounts[j] = ...
    ```
5. This write is out-of-bounds -> immediate revert.
6. splitTVS always fails, and the user loses gas.

### Impact

`_computeSplitArrays` is a core helper for splitting TVSs. Because the function always reverts, **no TVS can ever be split**. This breaks one of the fundamental mechanics in the IWO model (dynamic restructuring of vested positions).

Users attempting to split positions will always lose gas, and protocol functionality is effectively disabled for TVS management.

### PoC

**Note :** Before Running this POC make sure that your `FeesManager::calculateFeeAndNewAmountForOneTVS` looks like below. Because there already exists a different issue. It should be fixed in order to test this.

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
        
    function test_USER_SPLIT_REVERT_OUT_OF_BOUND_IN_computeSplitArrays()public{
        address[] memory KOLS=new address[](3);
        KOLS[0] = USER1;
        KOLS[1] = USER2;
        KOLS[2] = USER3;
        uint256 totalTVSAllocation = 9000e18;
        uint256 vestingPeriod=100;
        createProjectsAndSetAllocation(vestingPeriod,totalTVSAllocation,KOLS);

        uint256 rewardProjectId=0;
        uint256 user0NftId=nft.totalSupply()+1;
        hoax(OWNER);
        vesting.setTreasury(address(1));
        startHoax(USER1);
        vesting.claimRewardTVS(rewardProjectId);


        uint256[] memory percentages=new uint256[](2);
        percentages[0]=1000;
        percentages[1]=9000;
        uint256 splitNftId=user0NftId;
        vm.expectRevert("panic: array out-of-bounds access (0x32)");
        vesting.splitTVS(rewardProjectId, percentages, splitNftId);
    }
}
```

1. Add the above test in `test/vesting/TestAlignerzVesting.t.sol`
2. Run the following command in terminal.
```bash
forge test --mt test_USER_SPLIT_REVERT_OUT_OF_BOUND_IN_computeSplitArrays
```
3. You will see the output something simillar to below
```bash
Ran 1 test for test/vesting/TestAlignerzVesting.t.sol:TestAlignerzVesting
[PASS] test_USER_SPLIT_REVERT_OUT_OF_BOUND_IN_computeSplitArrays() (gas: 1040234)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.49ms (800.02µs CPU time)
```

### Mitigation

```diff
contract AlignerzVesting{
    .
    .
    .
    .
    function _computeSplitArrays(
        Allocation storage allocation,
        uint256 percentage,
        uint256 nbOfFlows
    )
        internal
        view
        returns (
            Allocation memory alloc
        )
    {
+       alloc.amounts= new uint256[](nbOfFlows);
+       alloc.vestingPeriods= new uint256[](nbOfFlows);
+       alloc.vestingStartTimes= new uint256[](nbOfFlows);
+       alloc.claimedSeconds= new uint256[](nbOfFlows);
+       alloc.claimedFlows= new bool[](nbOfFlows);
        uint256[] memory baseAmounts = allocation.amounts;
        uint256[] memory baseVestings = allocation.vestingPeriods;
        uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
        uint256[] memory baseClaimed = allocation.claimedSeconds;
        bool[] memory baseClaimedFlows = allocation.claimedFlows;
        alloc.assignedPoolId = allocation.assignedPoolId;
        alloc.token = allocation.token;
        for (uint256 j; j < nbOfFlows;) {
            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
            alloc.vestingPeriods[j] = baseVestings[j];
            alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
            alloc.claimedSeconds[j] = baseClaimed[j];
            alloc.claimedFlows[j] = baseClaimedFlows[j];
            unchecked {
                ++j;
            }
        }
    }
    .
    .
    .
}
```
  