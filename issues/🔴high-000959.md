# [000959] Uninitialized array results in DoS
  
  ### Summary

The [_computeSplitArrays()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113) returns an [Allocation](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L71) struct. This struct holds some arrays inside it, 5 arrays. In this function values are assigned into those arrays, but that  value assign operation reverts because those arrays were never initialized. 

### Root Cause

The _computeSplitArrays() returns an Allocation struct. 
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
```
This Allocation struct has few arrays inside it:
```solidity
struct Allocation {
        uint256[] amounts; // Amount of tokens committed for this allocation for all flows
        uint256[] vestingPeriods; // Chosen vesting duration in seconds for all flows
        uint256[] vestingStartTimes; // start time of the vesting for all flows
        uint256[] claimedSeconds; // Number of seconds already claimed for all flows
        bool[] claimedFlows; // Whether flow is claimed
        bool isClaimed; // Whether TVS is fully claimed
        IERC20 token; // The TVS token
        uint256 assignedPoolId; // Relevant for bidding projects: Id of the Pool (poolId=0; pool#1 / poolId=1; pool#2 /...) - for reward projects it will be 0 as default
    }
```
In the `_computeSplitArrays()` values are assigned to these arrays. But notice one thing that these arrays were never initialized. It means these arrays does not have any place inside it to hold a value yet. Without initializing these arrays it was tried to assign value into those:
```solidity
            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
            alloc.vestingPeriods[j] = baseVestings[j];
            alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
            alloc.claimedSeconds[j] = baseClaimed[j];
            alloc.claimedFlows[j] = baseClaimedFlows[j];
```
which results in out of bound array error. 

### Internal Pre-conditions

None.

### External Pre-conditions

Smart contract should be live. 

### Attack Path

None. 

### Impact

The `splitTVS()` call will revert. 

### PoC

To work this PoC correctly we need to fix a function before everything: the `FeesManager::calculateFeeAndNewAmountForOneTVS()`. Update this function to this:
```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint[](amounts.length);  // @audit-info this line added
        for (uint256 i; i < length; i++) {   // @audit-info i++ added
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            // @audit fee amount is compounding with each iteration
            newAmounts[i] = amounts[i] - feeAmount;
            //revert("checking point");
        }
    }
```
These 2 were added. I already submitted separate issues for each, why they should be added. If we don't update this function now then our call flow never reach the `_computeSplitArrays()`, as a result I can't show the issue. 

Now after updating the above mentioned function as I advised create a test file inside `./test` and paste this code:
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
Run this code:
```solidity
forge clean && forge build && forge test --mt test_splitTVS -vvvv
```
You will see this error:
```solidity
├─ [43841] ERC1967Proxy::fallback(0, [2000, 2000, 2000, 2000, 2000], 1)
    │   ├─ [43061] AlignerzVesting::splitTVS(0, [2000, 2000, 2000, 2000, 2000], 1) [delegatecall]
    │   │   ├─ [4318] AlignerzNFT::extOwnerOf(1) [staticcall]
    │   │   │   └─ ← [Return] KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30]
    │   │   ├─ [9944] Aligners26::transfer(ECRecover: [0x0000000000000000000000000000000000000001], 0)
    │   │   │   ├─ emit Transfer(from: ERC1967Proxy: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], to: ECRecover: [0x0000000000000000000000000000000000000001], value: 0)
    │   │   │   └─ ← [Return] true
    │   │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    └─ ← [Revert] panic: array out-of-bounds access (0x32)
```
The source of the error is [this](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1132) line:
```solidity
alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
```
To be more precise [all these](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1132-L1136) lines:
```solidity
            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
            alloc.vestingPeriods[j] = baseVestings[j];
            alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
            alloc.claimedSeconds[j] = baseClaimed[j];
            alloc.claimedFlows[j] = baseClaimedFlows[j];
```
Why? Because they were never initialized. 

### Mitigation

```diff
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
+        uint256 len = allocation.amounts.length;
+        alloc.amounts = new uint256[](len);
+        alloc.vestingPeriods = new uint256[](len);
+        alloc.vestingStartTimes = new uint256[](len);
+        alloc.claimedSeconds = new uint256[](len);
+        alloc.claimedFlows = new bool[](len);

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
Run the test again, you will see that worked perfectly. 
  