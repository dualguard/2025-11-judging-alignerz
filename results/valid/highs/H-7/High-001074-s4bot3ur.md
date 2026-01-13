# [001074] FeeManager::calculateFeeAndNewAmountForOneTVS will always make splitTVS and mergeTVS always revert due to infinite loop
  
  ### Summary

The missing iterator update in `FeeManager::calculateFeeAndNewAmountForOneTVS` causes the for loop to never progress, resulting in an infinite loop and **Out-of-Gas (OOG) revert** whenever the function is called. This makes any path that relies on this helper (such as `splitTVS` and `mergeTVS`) non-executable in practice: users calling these functions will have their transactions revert while still paying gas, leading to a **denial-of-service on core TVS management features**.

### Root Cause

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169

The above function uses for loop of the form:
```solidity
for (uint256 i; i < length;) {
    ...
}
```
but never updates i inside the loop body (i stays 0 forever), Leading to **Out Of Gas Revert**

### Internal Pre-conditions

1. splitTVS or mergeTVS is called (these call the helper).
2. amounts.length == length (expected) — the function assumes length describes the number of flows.
3. The function executes on-chain (no off-chain guard preventing it).

### External Pre-conditions

None required beyond normal protocol operation. Any normal call to `splitTVS / mergeTVS` that reaches `calculateFeeAndNewAmountForOneTVS` will trigger the issue.

### Attack Path

1. All the `AlignerzVesting` configuation has been set properly and a project is laucnhed.
2. User bids for large amount of tokens and Once bids finalize user will claim all tokens
3. User calls `splitTVS(...)` or `mergeTVS(...)` which internally calls calculateFeeAndNewAmountForOneTVS(...).
4. `calculateFeeAndNewAmountForOneTVS` enters its `for` loop and never updates the `i` after iteration is done
5. The entire transaction reverts — `splitTVS` / `mergeTVS` fails and no state changes occur.

### Impact

This bug makes the entire fee-calculation function revert every time, which means `splitTVS` and `mergeTVS` can never run successfully. 

Splitting and merging TVSs are central mechanics in the Alignerz model. If these operations are broken, **users cannot manage their TVSs as designed**; this degrades core functionality (operational risk) and may prevent valid user flows. **The documentation emphasises flexible TVS management**.

Also Users lose gas on every attempt to use these features.

### PoC

**Note :** Before Running this POC make sure that your `FeesManager::calculateFeeAndNewAmountForOneTVS` looks like below. Because there already exists a different issue. It should be fixed in order to test this.

```solidity
contract FeesManager{
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
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
        
    function test_TVS_SPLIT_REVERT_WITH_OUT_OF_GAS()public{
        uint256[] memory amounts=new uint256[](2);
        amounts[0]=1e18;
        amounts[1]=1e18;
        uint256 length=2;
        vm.expectRevert();
        vesting.calculateFeeAndNewAmountForOneTVS(0, amounts, length);
        
        // As a Result splitTVS will also Fail

        address[] memory KOLS=new address[](3);
        KOLS[0] = USER1;
        KOLS[1] = USER2;
        KOLS[2] = USER3;
        uint256 totalTVSAllocation = 9000e18;
        uint256 vestingPeriod=100;
        createProjectsAndSetAllocation(vestingPeriod,totalTVSAllocation,KOLS);

        uint256 rewardProjectId=0;
        uint256 user0NftId=nft.totalSupply()+1;
        startHoax(USER1);
        vesting.claimRewardTVS(rewardProjectId);


        uint256[] memory percentages=new uint256[](2);
        percentages[0]=1000;
        percentages[1]=9000;
        uint256 splitNftId=user0NftId;
        vm.expectRevert();
        vesting.splitTVS(rewardProjectId, percentages, splitNftId);
        vm.stopPrank();
    }
}
```

1. Add the above test in `test/vesting/TestAlignerzVesting.t.sol`
2. Run the following command in terminal.
```bash
forge test --mt test_TVS_SPLIT_REVERT_WITH_OUT_OF_GAS
```
3. You will see the output something simillar to below
```bash
Ran 1 test for test/vesting/TestAlignerzVesting.t.sol:TestAlignerzVesting
[PASS] test_TVS_SPLIT_REVERT_WITH_OUT_OF_GAS() (gas: 1073419835)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.67s (3.66s CPU time)
```

### Mitigation

```diff
abstract contract FeesManager is OwnableUpgradeable {
    .
    .
    .
    .
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
+           unchecked{
+               ++i;
+           }
        }
    }
    .
    .
}
```
  