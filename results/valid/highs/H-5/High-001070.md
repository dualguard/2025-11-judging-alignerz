# [001070] Incorrect Fee Calculation in calculateFeeAndNewAmountForOneTVS Causes Excessive User Loss in Both Split & Merge Operations
  
  ### Summary

The core issue is that `calculateFeeAndNewAmountForOneTVS` incorrectly subtracts cumulative fees from each flow instead of subtracting only that flow’s own fee.
Because `splitTVS` and `mergeTVS` depend on this function, every split or merge (which has flows greater than 1) results in excessive user balance loss, breaking the expected TVS behavior and violating the protocol's economic design.

**What actually happens**
Inside the loop:
```solidity
feeAmount += calculateFeeAmount(feeRate, amounts[i]);
newAmounts[i] = amounts[i] - feeAmount;
```
**Problem:**
- feeAmount is a running total (sum of previous fees).
- That cumulative total is subtracted from each individual element.

This means:
- The first element loses `fee_0`
- The second element loses `fee_0` + `fee_1`
- The third element loses `fee_0` + `fee_1` + `fee_2`

Correct behavior should be:
```solidity
fee_i = calculateFeeAmount(feeRate, amounts[i]);
newAmounts[i] = amounts[i] - fee_i;
feeAmount += fee_i; // accumulate only for treasury transfer
```


### Root Cause

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L172

The function uses a cumulative fee inside the amount subtraction:
```solidity
newAmounts[i] = amounts[i] - feeAmount;  // This is wrong, feeAmount is cumulative fee across flows
```
Instead of subtracting the individual fee.

Because of this incorrect subtraction, every user performing a split or merge (which has flows greater than 1) permanently loses more tokens than intended, violating the documented IWO mechanism.

### Internal Pre-conditions

1. A user owns a TVS NFT containing multiple flows (`amounts.length > 1`).
2. User calls:
    - `splitTVS() or`
    - `mergeTVS()`
3. Fee rate (`splitFeeRate` or `mergeFeeRate`) is greater than zero.

### External Pre-conditions

None.
This bug is purely internal and always triggers when the function is used.

### Attack Path

1. User invokes `splitTVS()` or `mergeTVS()` (normal behavior).
2. The contract calls `calculateFeeAndNewAmountForOneTVS`.
3. Each flow subtracts cumulative fees -> excessive token removal.
4. New TVS allocations store permanently reduced balances.
5. User suffers token loss

### Impact

This bug creates direct, permanent token loss for all users interacting with TVS splitting and merging.

- If a user splits a TVS with flows:
    Example amounts: `[3000, 3000, 3000]`
    Fee per flow: `50` (0.05 fee%)
    Expected result: each becomes `2985`.
    Actual result:
    - newAmount[0] = 3000 - 15 = 2985
    - newAmount[1] = 3000 - (15 + 15) = 2970
    - newAmount[2] = 3000 - (15 + 15 + 15) = 2955

    Total user loss = 2985 *3 - (2985 +2970 + 2955)
                    = 8955 - 8910
                    = 45 tokens (Additional loss)
This compounds with larger flows and larger fee rates. 

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
    
    function test_USER_EXPERIENCE_LOSS_WHEN_MERGE_OR_SPLITS_DUR_TO_INCORROECT_FEE_CALC()public{
        address[] memory KOLS=new address[](3);
        KOLS[0] = USER1;
        KOLS[1] = USER2;
        KOLS[2] = USER3;
        uint256 totalTVSAllocation = 9000e18;
        uint256 vestingPeriod=100;
        createProjectsAndSetAllocation(vestingPeriod,totalTVSAllocation,KOLS);
        uint256 rewardProjectId=0;
        uint256 startNftId=1;
        uint256 nextMergeNftId=startNftId+1;
        
        uint256 currentNftId=startNftId;
        for(uint8 i=0;i<KOLS.length;i++){
            hoax(KOLS[i]);
            vesting.claimRewardTVS(rewardProjectId);
            if(KOLS[i]!=USER1){
                hoax(KOLS[i]);
                nft.transferFrom(KOLS[i], USER1, currentNftId);
            }
            currentNftId++;
        }

        uint256[] memory projectIds=new uint256[](1);
        uint256[] memory nftIds=new uint256[](1);

        projectIds[0]=rewardProjectId;
        nftIds[0]=nextMergeNftId;
        nextMergeNftId+=1;

        hoax(OWNER);
        vesting.setTreasury(address(1));

        hoax(USER1);
        vesting.mergeTVS(rewardProjectId, startNftId, projectIds, nftIds);

        hoax(OWNER);
        vesting.setMergeFeeRate(50); // 0.5%

        nftIds[0]=nextMergeNftId;
        hoax(USER1);
        vesting.mergeTVS(rewardProjectId, startNftId, projectIds, nftIds);
        
        uint256 exectedAmountAfterFees=(((10000-50)*totalTVSAllocation )/10000);
        uint256 actualTokensReceived=vesting.getTotalTokens(rewardProjectId, startNftId, false);
        assert(exectedAmountAfterFees > actualTokensReceived);
        console.log("USER LOSS AMOUNT : %e",exectedAmountAfterFees -actualTokensReceived);
    }


}
```

1. Add the above test in `test/vesting/TestAlignerzVesting.t.sol`
2. Run the following command in terminal.
```bash
forge test --mt test_USER_EXPERIENCE_LOSS_WHEN_MERGE_OR_SPLITS_DUR_TO_INCORROECT_FEE_CALC -vv
```
3. You will see the output something simillar to below
```bash
Ran 1 test for test/vesting/TestAlignerzVesting.t.sol:TestAlignerzVesting
[PASS] test_USER_EXPERIENCE_LOSS_WHEN_MERGE_OR_SPLITS_DUR_TO_INCORROECT_FEE_CALC() (gas: 2364755)
Logs:
  USER LOSS AMOUNT : 1.5e19

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.16ms (2.18ms CPU time)
```


### Mitigation

```diff

abstract contract FeesManager is OwnableUpgradeable {
    .
    .
    .
    .
-   function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+   function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 totalFeeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
-           feeAmount += calculateFeeAmount(feeRate, amounts[i]);
+           feeAmount = calculateFeeAmount(feeRate, amounts[i]);
+           totalFeeAmount+=feeAmount;
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
    .
    .
    .
}
```
  