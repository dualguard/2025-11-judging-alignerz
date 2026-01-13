# [001072] Original TVS Allocation Is Permanently Overwritten During splitTVS, Breaking All Subsequent Logic
  
  ### Summary

In `AlignerzVesting::splitTVS`, when i == 0, the contract reuses the original NFT ID (splitNftId) as the target for writing split results.
Because of this:
1. The function computes a partial-split allocation using _computeSplitArrays(...).
2. It then assigns this **partial allocation back into the original NFT**, via:
```solidity
uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);
Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
Allocation storage newAlloc = ... allocations[nftId];
_assignAllocation(newAlloc, alloc);
```
Since `nftId == splitNftId` when `i == 0`, the contract overwrites the original Allocation storage with only the first percentage-split, permanently destroying data:

Assume a user has 3000 tokens inside their original TVS NFT and wants to split:
- 10% remains in the original NFT
- 90% goes to a new NFT

What should happen (correct behavior):
- Original NFT:
    `3000 * 10% = 300`
- New NFT:
    `3000 * 90% = 2700`
Total after split = 300 + 2700 = 3000 (no loss, considering fee as zero)

**What actually happens due to the bug**
Because the contract overwrites the original allocation in the first iteration:
**Iteration 1 (i = 0, using splitNftId)**
```solidity
new_amount = 3000 * 10% = 300
```
This 300 replaces the original 3000 inside storage.
The original amount is now lost forever.
**Iteration 2 (i = 1)**
Instead of using the original 3000 again, the calculation incorrectly uses the already overwritten 300:
```solidity
new_amount = 300 * 90% = 270
```
So the final allocations become:
- Original NFT: **300**
- New NFT: **270**
- Total after split = **570**

Total Loss for user in this case is **2430** tokens.

### Root Cause

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1086

In the above line for the first iteration it will assign `nftId` as `splitNftId` (original nft). For other iterations it will use the new minted tokenid

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1088

This will get the percentage of amounts that the current `nftId` must receive. 

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1091

This will write the new percentage amounts to respective nfts. The issue here is when `nftId` is `splitNftId` it will overwrite the previous amounts of the original `splitNftId`. In the next iterations this updated value will be used to calculate split amounts for new nfts

### Internal Pre-conditions

1. User owns a TVS NFT with at least 1 flow.
2. User calls `splitTVS(projectId, percentages, splitNftId)` with `percentages.length ≥ 2`.
3. The first iteration `(i == 0)` triggers the overwrite.

### External Pre-conditions

None
This vulnerability occurs for every call to `splitTVS`.

### Attack Path

1. User attempts to split a TVS NFT into two or more percentages.
2. On first iteration (i == 0):
    - `_computeSplitArrays` produces a partial allocation (e.g., 30% of the original).

    - That partial allocation is **written directly into the original NFT**.
3. The original allocation is now corrupted, containing only a fraction of its true values.
4. Next iterations compute splits based on this corrupted partial data.
5. Final output allocations are mathematically incorrect, and vesting logic becomes permanently corrupted.
6. User loses the remaining portions of their original allocation.

### Impact

This vulnerability will lead to direct loss for users when they tries to split their TVS


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

```solidity
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
        alloc.amounts= new uint256[](nbOfFlows);
        alloc.vestingPeriods= new uint256[](nbOfFlows);
        alloc.vestingStartTimes= new uint256[](nbOfFlows);
        alloc.claimedSeconds= new uint256[](nbOfFlows);
        alloc.claimedFlows= new bool[](nbOfFlows);
        uint256[] memory baseAmounts = allocation.amounts;
        uint256[] memory baseVestings = allocation.vestingPeriods;
        uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
        uint256[] memory baseClaimed = allocation.claimedSeconds;
        bool[] memory baseClaimedFlows = allocation.claimedFlows;
        alloc.assignedPoolId = allocation.assignedPoolId;
        alloc.token = allocation.token;
        for (uint256 j; j < nbOfFlows;) {
            //@audit look for precision los
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

    function getTotalTokens(uint256 nftId)public view returns(uint256 amount){
        uint256 len =allocationOf[nftId].amounts.length;
        for(uint256 i=0;i<len;i++){
            amount+=allocationOf[nftId].amounts[i];
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
    
    function test_USER_EXPERIENCE_LOSS_WHEN_SPLITS()public{
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
        
        uint256 totalAmountsBeforeSplit=vesting.getTotalTokens(splitNftId);
        uint256 newnftId=nft.totalSupply()+1;
        vesting.splitTVS(rewardProjectId, percentages, splitNftId);
        uint256 totalAmountsAfterSplit=vesting.getTotalTokens(splitNftId)+vesting.getTotalTokens(newnftId);
        assert(totalAmountsAfterSplit < totalAmountsBeforeSplit);

        uint256 amountLeftInOldNFT=(totalAmountsBeforeSplit * percentages[0])/10000;
        uint256 amountLeftInNewNFT=(amountLeftInOldNFT * percentages[1])/10000;
        assert(totalAmountsAfterSplit== amountLeftInOldNFT+ amountLeftInNewNFT);
        console.log("Amount Loss To User :",totalAmountsBeforeSplit - totalAmountsAfterSplit);
    }


}
```

1. Add the above test in `test/vesting/TestAlignerzVesting.t.sol`
2. Run the following command in terminal.
```bash
forge test --mt test_USER_EXPERIENCE_LOSS_WHEN_SPLITS
```
3. You will see the output something simillar to below
```bash
Ran 1 test for test/vesting/TestAlignerzVesting.t.sol:TestAlignerzVesting
[PASS] test_USER_EXPERIENCE_LOSS_WHEN_SPLITS() (gas: 1583819)
Logs:
  Amount Loss To User : 2430000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.29ms (1.18ms CPU time)
```

### Mitigation

_No response_
  