# [000958] TVS is not splitting with correct amount due to logic flaw in AlignerzVesting::splitTVS()
  
  ### Summary

When user calls the splitTVS() to split a TVS into few percentages, then all TVSs, after splitting, should get equal amount. For ex- if I want to split my TVS for 5 percentages, i.e 2000,2000,2000,2000 & 2000, & if the amount allocated for the main TVS is 1000e18 then all 5 TVSs will get 200e18 amount at the end. But the logic flaw in this function does not let this happen. 

### Root Cause

Look at [these](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1086-L1091) lines:
```solidity
        uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);
        if (i != 0) newNftIds[i - 1] = nftId;
        Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
        NFTBelongsToBiddingProject[nftId] = isBiddingProject ? true : false;
        Allocation storage newAlloc = isBiddingProject ? biddingProjects[projectId].allocations[nftId] : rewardProjects[projectId].allocations[nftId];
        _assignAllocation(newAlloc, alloc);
```
Now assume we want to split a RewardProject NFT into 5 percentages, 2000,2000,2000,2000 & 2000, just as I said in summary. 
Here the _computeSplitArrays() returning the splitted version of Allocation for the percentage 2000. 

Now the very first iteration of the [for loop](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1082) i ==0, so as i == 0 the nftId will be splitNftId. After that the _computeSplitArrays() returned an Allocation which has 2000  in amounts[0]. After that this line executes:
```solidity
Allocation storage newAlloc = isBiddingProject ? biddingProjects[projectId].allocations[nftId] : rewardProjects[projectId].allocations[nftId];
```
Note that here the nftId is the main TVS's id. Here we fetched the main TVS's allocation as newAlloc. 
Now just after that the code is calling _assignAllocation(), where the result Allocation from _computeSplitArrays() will be written on newAlloc. i.e the Allocation of the main TVS will be overwritten by the resulted Allocation of _computeSplitArrays(). Means after the _assignAllocation() call the amounts[0] of the main TVSs will be 200e18. Fine, not any issue till now. But in next all iterations this main TVSs Allocation is passed to _computeSplitArrays(), it means the splitting is happening now on 200e18, not the main amount of 1000e18. It means all further 4 TVSs will receive 40e18 as amount instead of 200e18. 

### Internal Pre-conditions

None. 

### External Pre-conditions

None. 

### Attack Path

None.

### Impact

All further NFTs except the main NFT will get less amount of allocation. 

### PoC

Before the running the test successfully we need to update 2 functions, replace the  [FeesManager::calculateFeeAndNewAmountForOneTVS()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169) with this:
```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint[](amounts.length);  // @audit-info this line added
        for (uint256 i; i < length; i++) {   // @audit-info i++ added
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);            
            newAmounts[i] = amounts[i] - feeAmount;        
        }
    }
```
and replace the [AlignerzVesting::_computeSplitArrays()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113) with this:
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
        uint len = allocation.amounts.length; // added
        alloc.amounts = new uint256[](len); // added
        alloc.vestingPeriods = new uint256[](len); // added
        alloc.vestingStartTimes = new uint256[](len); // added
        alloc.claimedSeconds = new uint256[](len); // added
        alloc.claimedFlows = new bool[](len); // added

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
```
The reason behind this update is these 2 functions has bugs, and these are the fixed version, if we don't fix the functions I can't show the working PoC. I have submitted reports related to these issues, you can find those with these titles: Uninitialized array results in DoS, Infinite loop will result in out of gas error i.e DoS and Flawed logic in FeesManager::calculateFeeAndNewAmountForOneTVS() will result DoS. For now just update those functions. 
Now create a test file under ./test directory and paste this code:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "./AlignerzVestingProtocolTest.t.sol";

contract RewardProjectTest is AlignerzVestingProtocolTest {
        function test_splitTVS() public {
        uint currentTime = block.timestamp;
        vm.prank(projectCreator);
        // Launching reward project
        vesting.launchRewardProject(
            address(token),
            address(usdt),
            currentTime,
            30 days  // claimWindow
        );

        // Setting TVS allocation
        address[] memory kolTVS = new address[](1);
        uint[] memory TVSamounts = new uint[](1);

        kolTVS[0] = makeAddr("KOL-1");
        
        TVSamounts[0] = 1000e18;
        
        vm.prank(projectCreator);
        vesting.setTVSAllocation(0, 1000e18, 60 days, kolTVS, TVSamounts);

        vm.warp(currentTime + 25 days);

        vm.prank(makeAddr("KOL-1"));
        vesting.claimRewardTVS(0);

        assertEq(nft.extOwnerOf(1), makeAddr("KOL-1"));

        uint[] memory percentages = new uint[](5);
        percentages[0] = 2000;
        percentages[1] = 2000;
        percentages[2] = 2000;
        percentages[3] = 2000;
        percentages[4] = 2000;
        assertEq(vesting.splitFeeRate(),0,"Split fee error");
        vm.prank(makeAddr("KOL-1"));
        vesting.splitTVS(0,percentages,1);
    }
}
```
Run the test using this command:
```solidity
forge clean && forge build
forge test --mt test_splitTVS -vvvv
```
The output is this:
```solidity
├─ [2045587] ERC1967Proxy::fallback(0, [2000, 2000, 2000, 2000, 2000], 1)
    │   ├─ [2044793] AlignerzVesting::splitTVS(0, [2000, 2000, 2000, 2000, 2000], 1) [delegatecall]
    │   │   ├─ [4318] AlignerzNFT::extOwnerOf(1) [staticcall]
    │   │   │   └─ ← [Return] KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30]
    │   │   ├─ [9944] Aligners26::transfer(ECRecover: [0x0000000000000000000000000000000000000001], 0)
    │   │   │   ├─ emit Transfer(from: ERC1967Proxy: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], to: ECRecover: [0x0000000000000000000000000000000000000001], value: 0)
    │   │   │   └─ ← [Return] true
    │   │   ├─ emit TVSSplit(projectId: 0, isBiddingProject: false, splitNftId: 1, nftId: 1, amounts: [200000000000000000000 [2e20]], vestingPeriods: [5184000 [5.184e6]], vestingStartTimes: [1], claimedSeconds: [0])
    │   │   ├─ [35488] AlignerzNFT::mint(KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30])
    │   │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30], tokenId: 2)
    │   │   │   ├─ emit Minted(to: KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30], tokenId: 2)
    │   │   │   └─ ← [Return] 2
    │   │   ├─ emit TVSSplit(projectId: 0, isBiddingProject: false, splitNftId: 1, nftId: 2, amounts: [40000000000000000000 [4e19]], vestingPeriods: [5184000 [5.184e6]], vestingStartTimes: [1], claimedSeconds: [0])
    │   │   ├─ [35488] AlignerzNFT::mint(KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30])
    │   │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30], tokenId: 3)
    │   │   │   ├─ emit Minted(to: KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30], tokenId: 3)
    │   │   │   └─ ← [Return] 3
    │   │   ├─ emit TVSSplit(projectId: 0, isBiddingProject: false, splitNftId: 1, nftId: 3, amounts: [40000000000000000000 [4e19]], vestingPeriods: [5184000 [5.184e6]], vestingStartTimes: [1], claimedSeconds: [0])
    │   │   ├─ [35488] AlignerzNFT::mint(KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30])
    │   │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30], tokenId: 4)
    │   │   │   ├─ emit Minted(to: KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30], tokenId: 4)
    │   │   │   └─ ← [Return] 4
    │   │   ├─ emit TVSSplit(projectId: 0, isBiddingProject: false, splitNftId: 1, nftId: 4, amounts: [40000000000000000000 [4e19]], vestingPeriods: [5184000 [5.184e6]], vestingStartTimes: [1], claimedSeconds: [0])
    │   │   ├─ [35488] AlignerzNFT::mint(KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30])
    │   │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30], tokenId: 5)
    │   │   │   ├─ emit Minted(to: KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30], tokenId: 5)
    │   │   │   └─ ← [Return] 5
    │   │   ├─ emit TVSSplit(projectId: 0, isBiddingProject: false, splitNftId: 1, nftId: 5, amounts: [40000000000000000000 [4e19]], vestingPeriods: [5184000 [5.184e6]], vestingStartTimes: [1], claimedSeconds: [0])
    │   │   └─ ← [Return] 1, [2, 3, 4, 5]
    │   └─ ← [Return] 1, [2, 3, 4, 5]
    └─ ← [Return]
```
Look at the amounts[], for nftId 1 it is 2e20 i.e 200e18, but for the last 4 nfts it is 4e19 i.e 40e18. 

### Mitigation

The mitigation contains 3 steps:

1. Take the snapshot of main TVS's Allocation as memory just after deducting the split fee. 
2. Pass the snapshot to _computeSplitArrays().
3. Update the allocation argument of _computeSplitArrays() from storage to memory. 

So it will look like these:
1. 
```diff
function splitTVS(
        uint256 projectId,
        uint256[] calldata percentages,
        uint256 splitNftId
    ) external returns (uint256, uint256[] memory) {
        address nftOwner = nftContract.extOwnerOf(splitNftId);
        require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());

        bool isBiddingProject = NFTBelongsToBiddingProject[splitNftId];
        (Allocation storage allocation, IERC20 token) = isBiddingProject ?
        (biddingProjects[projectId].allocations[splitNftId], biddingProjects[projectId].token) :
        (rewardProjects[projectId].allocations[splitNftId], rewardProjects[projectId].token);

        uint256[] memory amounts = allocation.amounts;
        uint256 nbOfFlows = allocation.amounts.length;
        (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);
        allocation.amounts = newAmounts;
        token.safeTransfer(treasury, feeAmount);

        uint256 nbOfTVS = percentages.length;
+      Allocation memory baseAllocation = allocation;     // Snapshot
        
         // Rest of the code
         
          for (uint256 i; i < nbOfTVS;) {
            uint256 percentage = percentages[i];
            sumOfPercentages += percentage;

            uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);
            if (i != 0) newNftIds[i - 1] = nftId;
-           Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
+           Allocation memory alloc = _computeSplitArrays( baseAllocation, percentage, nbOfFlows);
             // Rest of the code
            }
            // Rest of the code
        }
```
2.
```diff
function _computeSplitArrays(
-       Allocation storage allocation,
+       Allocation memory allocation, 
        uint256 percentage,
        uint256 nbOfFlows
    )
        internal
        view
        returns (
            Allocation memory alloc
        )
    {
       // Rest of the code
     } 
```
After updating the code as mentioned run the test again, you will see this output now:
```solidity
├─ [2022214] ERC1967Proxy::fallback(0, [2000, 2000, 2000, 2000, 2000], 1)
    │   ├─ [2021420] AlignerzVesting::splitTVS(0, [2000, 2000, 2000, 2000, 2000], 1) [delegatecall]
    │   │   ├─ [4318] AlignerzNFT::extOwnerOf(1) [staticcall]
    │   │   │   └─ ← [Return] KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30]
    │   │   ├─ [9944] Aligners26::transfer(ECRecover: [0x0000000000000000000000000000000000000001], 0)
    │   │   │   ├─ emit Transfer(from: ERC1967Proxy: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], to: ECRecover: [0x0000000000000000000000000000000000000001], value: 0)
    │   │   │   └─ ← [Return] true
    │   │   ├─ emit TVSSplit(projectId: 0, isBiddingProject: false, splitNftId: 1, nftId: 1, amounts: [200000000000000000000 [2e20]], vestingPeriods: [5184000 [5.184e6]], vestingStartTimes: [1], claimedSeconds: [0])
    │   │   ├─ [35488] AlignerzNFT::mint(KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30])
    │   │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30], tokenId: 2)
    │   │   │   ├─ emit Minted(to: KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30], tokenId: 2)
    │   │   │   └─ ← [Return] 2
    │   │   ├─ emit TVSSplit(projectId: 0, isBiddingProject: false, splitNftId: 1, nftId: 2, amounts: [200000000000000000000 [2e20]], vestingPeriods: [5184000 [5.184e6]], vestingStartTimes: [1], claimedSeconds: [0])
    │   │   ├─ [35488] AlignerzNFT::mint(KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30])
    │   │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30], tokenId: 3)
    │   │   │   ├─ emit Minted(to: KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30], tokenId: 3)
    │   │   │   └─ ← [Return] 3
    │   │   ├─ emit TVSSplit(projectId: 0, isBiddingProject: false, splitNftId: 1, nftId: 3, amounts: [200000000000000000000 [2e20]], vestingPeriods: [5184000 [5.184e6]], vestingStartTimes: [1], claimedSeconds: [0])
    │   │   ├─ [35488] AlignerzNFT::mint(KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30])
    │   │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30], tokenId: 4)
    │   │   │   ├─ emit Minted(to: KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30], tokenId: 4)
    │   │   │   └─ ← [Return] 4
    │   │   ├─ emit TVSSplit(projectId: 0, isBiddingProject: false, splitNftId: 1, nftId: 4, amounts: [200000000000000000000 [2e20]], vestingPeriods: [5184000 [5.184e6]], vestingStartTimes: [1], claimedSeconds: [0])
    │   │   ├─ [35488] AlignerzNFT::mint(KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30])
    │   │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30], tokenId: 5)
    │   │   │   ├─ emit Minted(to: KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30], tokenId: 5)
    │   │   │   └─ ← [Return] 5
    │   │   ├─ emit TVSSplit(projectId: 0, isBiddingProject: false, splitNftId: 1, nftId: 5, amounts: [200000000000000000000 [2e20]], vestingPeriods: [5184000 [5.184e6]], vestingStartTimes: [1], claimedSeconds: [0])
    │   │   └─ ← [Return] 1, [2, 3, 4, 5]
    │   └─ ← [Return] 1, [2, 3, 4, 5]
    └─ ← [Return]
```
You can see all TVSs are getting equal amount now. 
  