# [000281] `FeesManager::calculateFeeAndNewAmountForOneTVS` Always Reverts Due to Infinite Loop & Incorrect Array Creation, Leading to Users Unable to Merge TVS
  
  ### Summary

The `FeesManager::calculateFeeAndNewAmountForOneTVS` function has two separate issues that cause it to always revert. This means that users are never able to merge their TVS NFTs. 

### Root Cause

If a user has more than one TVS NFTs they can choose to merge their NFTs into a single one. 
In order to complete that they call the following function:

```solidity
function mergeTVS(
        uint256 projectId,
        uint256 mergedNftId,
        uint256[] calldata projectIds,
        uint256[] calldata nftIds
    ) external returns (uint256) {
        address nftOwner = nftContract.extOwnerOf(mergedNftId);
        require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());

        bool isBiddingProject = NFTBelongsToBiddingProject[mergedNftId]; 
        (Allocation storage mergedTVS, IERC20 token) = isBiddingProject
            ? (
                biddingProjects[projectId].allocations[mergedNftId],
                biddingProjects[projectId].token
            )
            : (
                rewardProjects[projectId].allocations[mergedNftId],
                rewardProjects[projectId].token
            );
        uint256[] memory amounts = mergedTVS.amounts;
        uint256 nbOfFlows = mergedTVS.amounts.length;
        (
            uint256 feeAmount,
            uint256[] memory newAmounts
@>      ) = calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);      <@

        mergedTVS.amounts = newAmounts;

        uint256 nbOfNFTs = nftIds.length;
        require(nbOfNFTs > 0, Not_Enough_TVS_To_Merge());
        require(nbOfNFTs == projectIds.length, Array_Lengths_Must_Match());

        for (uint256 i; i < nbOfNFTs; i++) {
            feeAmount += _merge(mergedTVS, projectIds[i], nftIds[i], token);
        }
        token.safeTransfer(treasury, feeAmount); 
        emit TVSsMerged(
            projectId,
            isBiddingProject,
            nftIds,
            mergedNftId,
            mergedTVS.amounts,
            mergedTVS.vestingPeriods,
            mergedTVS.vestingStartTimes,
            mergedTVS.claimedSeconds,
            mergedTVS.claimedFlows
        );
        return mergedNftId;
    }
```

As marked above, the `calculateFeeAndNewAmountForOneTVS()` is a helper function in the `FeesManager` abstract contract, which `AlignerzVesting` inherits, to calculate the fees and new amounts, accumulating all the NFTs info into a single NFT. 

However, there are two bugs in this function. 

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) { <@
        for (uint256 i; i < length;) {    <@
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

1. It does not increment the `i` in the for loop. This means this becomes an infinite loop and will always revert with: `EvmError: OutOfGas`.

2. `newAmounts` array declaration is wrong. Solidity does not allow this type of declaration of arrays. Once the function hits this line `newAmounts[i] = amounts[i] - feeAmount;`, it will revert with: `panic: array out-of-bounds access (0x32)`. 

### Internal Pre-conditions

1. A user has to have more than 1 minted TVS NFTs.
2. A user has to choose to merge the NFTs into one.

### External Pre-conditions

N/A

### Attack Path

This is a bug in the contract that will prevent the merging of NFTs. No attack path here.

### Impact

**Impact:** Medium <br>
An entire section of functionality in the codebase is never working. No risk for funds. 

**Likelihood:** High <br>
This will happen every time a user wants to merge their NFTs. 

### PoC

Paste the following in the `AlignerzVestingProtocolTest.t.sol` test suite:

1. Run the test. It will pass.
2. Fix the contract with adding the incrementing of `i` in the loop and then you will get the infinite loop revert.

```solidity
function test_merging_tvs_reverts() public {
        // 1. Owner launches a reward project]
        vm.startPrank(projectCreator);
        uint256 rewardProjectID = vesting.rewardProjectCount();
        uint256 claimAndVestingPeriod = 60 days; // Same amount of time for simplicity
        vesting.launchRewardProject(
            address(token),
            address(usdt),
            block.timestamp,
            claimAndVestingPeriod
        );

        // 2. Owner sets up allocations for the reward project
        uint256 tvsAllocation = 3_000_000 ether;
        address[] memory kols = new address[](10);
        uint256[] memory kolAmounts = new uint256[](10);
        address kolAddress;
        uint256 totalAllocated = 0;
        for (uint256 i = 0; i < kols.length; i++) {
            kolAddress = makeAddr(string.concat("kol", vm.toString(i)));
            kols[i] = kolAddress;
            kolAmounts[i] = tvsAllocation / (kols.length * (i + 1));
            totalAllocated += kolAmounts[i];
        }
        if (tvsAllocation > totalAllocated)
            kolAmounts[0] += tvsAllocation - totalAllocated;
        vesting.setTVSAllocation(
            rewardProjectID,
            tvsAllocation,
            claimAndVestingPeriod,
            kols,
            kolAmounts
        );

        vm.stopPrank();

        // 3. KOLS can claim their TVS
        for (uint256 i = 0; i < kols.length; i++) {
            vm.prank(kols[i]);

            vesting.claimRewardTVS(rewardProjectID);
        }

        // Merge TVS with one
        uint256[] memory projectIdsArray = new uint256[](1);
        uint256[] memory nftIdsArray = new uint256[](1);
        projectIdsArray[0] = rewardProjectID;
        nftIdsArray[0] = 1;

        vm.prank(kols[0]);
        vm.expectRevert(stdError.indexOOBError);
        vesting.mergeTVS(rewardProjectID, 1, projectIdsArray, nftIdsArray);
    }
```

### Mitigation

Increment the `i` in the for loop and declare the array correctly:

```diff
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+       newAmounts = new uint256[](amounts.length);
-       for (uint256 i; i < length;) 
+       for (uint256 i; i < length; i++) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
``` 
  