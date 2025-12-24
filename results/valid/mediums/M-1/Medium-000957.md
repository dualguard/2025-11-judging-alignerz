# [000957] Outdated state while merging TVS result is loss of dividends
  
  ### Summary

In [AlignerzVesting::mergeTVS()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002) after merging TVSs one important state is not updated, the [allocationOf](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L114) mapping. As dividend calculation is depended on the value of this mapping, to be specific the amounts[], as allocationOf mapping returns an Allocation struct for a NFT id, there will be a huge loss of dividend for a user. 

### Root Cause

As mentioned in summary, the root cause of the issue is the allocationOf mapping is not updated for the given NFT/TVS. 
For ex- 
A user did split his TVS into 5, for 5 different percentage, let say 2000 percentage each, amounts[] was 1000e18 so it will have 200e18 each. After splitting main TVS's Allocation.amounts[] will have 200e18. After split the allocationOf mapping for main nft/TVS is set to current value, i.e amounts[] will have 200e18, let say the main TVS id is 1. So allocationOf(1) returns a Allocation which has amounts[200e18]. You can see the exact line where allocationOf mapping stores the data [here](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1092).
Now user decided to merge those TVSs again, after merging the final Allocation will have updated amounts[] i.e amounts[200e18,200e18,200e18,200e18,200e18] for the main TVS id i.e 1. But note that the merge function i.e mergeTVS() does not update the allocationOf mapping for nft id 1, it is still the old one. 
When dividend is calculated the allocationOf mapping is read, see here: [1](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L142), [2](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L150), [3](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L160). As the allocationOf mapping is still outdated i.e the amounts[] of Allocation of that nft id only holds one element which is 200e18 instead of five 200e18, so only 200e18 will be counted instead of 1000e18. Means the user will loss a lot of dividend. 

### Internal Pre-conditions

None.

### External Pre-conditions

None.

### Attack Path

none.

### Impact

Loss of dividend. 

### PoC

Before the running the test successfully we need to update 2 functions, replace the [FeesManager::calculateFeeAndNewAmountForOneTVS()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169) with this:
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
For testing purpose, to fetch the complete Allocation from allocationOf mapping for an nft/TVS i added a function in AlignerzVesting contract, add the function to properly track the tx:
```solidity
function getFullAllocation(uint nftId) public view returns(Allocation memory allocation){
        allocation = allocationOf[nftId];
    }
```
Now create a test file under ./test directory and paste this code:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "./AlignerzVestingProtocolTest.t.sol";

contract RewardProjectTest is AlignerzVestingProtocolTest {
            function test_mergeTVS() public {

        uint currentTime = block.timestamp;
        vm.prank(projectCreator);
        // Launching reward project
        vesting.launchRewardProject(
            address(token),
            address(usdt),
            currentTime,
            30 days // claimWindow
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
         
        // Split the NFT
        uint[] memory percentages = new uint[](5);
        percentages[0] = 2000;
        percentages[1] = 2000;
        percentages[2] = 2000;
        percentages[3] = 2000;
        percentages[4] = 2000;
        assertEq(vesting.splitFeeRate(), 0, "Split fee error");
        //confirming that allocation.amounts[] are correct
        // uint[] memory amounts_ = vesting.getAllocationAmountsArray(1);
        // assertEq(amounts_.length,1);
        // assertEq(amounts_[0],1000e18);
        vm.prank(makeAddr("KOL-1"));
        vesting.splitTVS(0, percentages, 1);
        // uint[] memory amounts_ = vesting.getAllocationAmountsArray(1);
        // assertEq(amounts_[0],200e18);

        // Lets merge now
        uint[] memory projectIds_ = new uint[](4);
        projectIds_[0] = 0;
        projectIds_[1] = 0;
        projectIds_[2] = 0;
        projectIds_[3] = 0;

        uint[] memory nftIds_ = new uint[](4);
        nftIds_[0] = 2;
        nftIds_[1] = 3;
        nftIds_[2] = 4;
        nftIds_[3] = 5;

        vm.prank(makeAddr("KOL-1"));
        uint newNftId = vesting.mergeTVS(0, 1, projectIds_, nftIds_);
        console.log("Merged nftId = ", newNftId);
        vesting.getFullAllocation(1);      
    }
}
```
Run the test:
```solidity
forge clean && forge build && forge test --mt test_mergeTVS -vvvv
```
Output:
```solidity
│   │   ├─ emit TVSsMerged(projectId: 0, isBiddingProject: false, nftIds: 0xa92fddc9a12b24044408528c7026f0de38c003a1d354be3e5326c376003956d9, mergedNftId: 1, amounts: [200000000000000000000 [2e20], 200000000000000000000 [2e20], 200000000000000000000 [2e20], 200000000000000000000 [2e20], 200000000000000000000 [2e20]], vestingPeriods: [5184000 [5.184e6], 5184000 [5.184e6], 5184000 [5.184e6], 5184000 [5.184e6], 5184000 [5.184e6]], vestingStartTimes: [1, 1, 1, 1, 1], claimedSeconds: [0, 0, 0, 0, 0], claimedFlows: [false, false, false, false, false])
    │   │   └─ ← [Return] 1
    │   └─ ← [Return] 1
    ├─ [0] console::log("Merged nftId = ", 1) [staticcall]
    │   └─ ← [Stop]
    ├─ [11977] ERC1967Proxy::fallback(1) [staticcall]
    │   ├─ [11144] AlignerzVesting::getFullAllocation(1) [delegatecall]
    │   │   └─ ← [Return] Allocation({ amounts: [200000000000000000000 [2e20]], vestingPeriods: [5184000 [5.184e6]], vestingStartTimes: [1], claimedSeconds: [0], claimedFlows: [false], isClaimed: false, token: 0x2e234DAe75C793f67A35089C9d99245E1C58470b, assignedPoolId: 0 })
    │   └─ ← [Return] Allocation({ amounts: [200000000000000000000 [2e20]], vestingPeriods: [5184000 [5.184e6]], vestingStartTimes: [1], claimedSeconds: [0], claimedFlows: [false], isClaimed: false, token: 0x2e234DAe75C793f67A35089C9d99245E1C58470b, assignedPoolId: 0 })
    └─ ← [Return]
```
You can see even if the TVSsMerged event was emitted with valid data but the allocationOf mapping is not returning the updated Allocation data for the given NFT id. Because the mapping was not updated. 

### Mitigation

```diff
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
+      allocationOf[mergedNftId] = mergedTVS;
        emit TVSsMerged(projectId, isBiddingProject, nftIds, mergedNftId, mergedTVS.amounts, mergedTVS.vestingPeriods, mergedTVS.vestingStartTimes, mergedTVS.claimedSeconds, mergedTVS.claimedFlows);
        return mergedNftId;
    }
```
Now run the test again, you will see this output:
```solidity
│   │   ├─ emit TVSsMerged(projectId: 0, isBiddingProject: false, nftIds: 0xa92fddc9a12b24044408528c7026f0de38c003a1d354be3e5326c376003956d9, mergedNftId: 1, amounts: [200000000000000000000 [2e20], 200000000000000000000 [2e20], 200000000000000000000 [2e20], 200000000000000000000 [2e20], 200000000000000000000 [2e20]], vestingPeriods: [5184000 [5.184e6], 5184000 [5.184e6], 5184000 [5.184e6], 5184000 [5.184e6], 5184000 [5.184e6]], vestingStartTimes: [1, 1, 1, 1, 1], claimedSeconds: [0, 0, 0, 0, 0], claimedFlows: [false, false, false, false, false])
    │   │   └─ ← [Return] 1
    │   └─ ← [Return] 1
    ├─ [0] console::log("Merged nftId = ", 1) [staticcall]
    │   └─ ← [Stop]
    ├─ [24064] ERC1967Proxy::fallback(1) [staticcall]
    │   ├─ [23109] AlignerzVesting::getFullAllocation(1) [delegatecall]
    │   │   └─ ← [Return] Allocation({ amounts: [200000000000000000000 [2e20], 200000000000000000000 [2e20], 200000000000000000000 [2e20], 200000000000000000000 [2e20], 200000000000000000000 [2e20]], vestingPeriods: [5184000 [5.184e6], 5184000 [5.184e6], 5184000 [5.184e6], 5184000 [5.184e6], 5184000 [5.184e6]], vestingStartTimes: [1, 1, 1, 1, 1], claimedSeconds: [0, 0, 0, 0, 0], claimedFlows: [false, false, false, false, false], isClaimed: false, token: 0x2e234DAe75C793f67A35089C9d99245E1C58470b, assignedPoolId: 0 })
    │   └─ ← [Return] Allocation({ amounts: [200000000000000000000 [2e20], 200000000000000000000 [2e20], 200000000000000000000 [2e20], 200000000000000000000 [2e20], 200000000000000000000 [2e20]], vestingPeriods: [5184000 [5.184e6], 5184000 [5.184e6], 5184000 [5.184e6], 5184000 [5.184e6], 5184000 [5.184e6]], vestingStartTimes: [1, 1, 1, 1, 1], claimedSeconds: [0, 0, 0, 0, 0], claimedFlows: [false, false, false, false, false], isClaimed: false, token: 0x2e234DAe75C793f67A35089C9d99245E1C58470b, assignedPoolId: 0 })
    └─ ← [Return]
```
  