# [000032] `alloc` arrays are not initialized in `_computeSplitArrays` blocking splitting TVS
  
  ### Summary

`_computeSplitArrays` is called when splitting TVS positions, which is used to split the original amount according to the passed percentage. This returns an `alloc` with all the split amounts. However, none of the arrays in the allocation are initialized:
```solidity
function _computeSplitArrays(
    Allocation memory allocation,
    uint256 percentage,
    uint256 nbOfFlows
) internal view returns (Allocation memory alloc) {
    uint256[] memory baseAmounts = allocation.amounts;
    uint256[] memory baseVestings = allocation.vestingPeriods;
    uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
    uint256[] memory baseClaimed = allocation.claimedSeconds;
    bool[] memory baseClaimedFlows = allocation.claimedFlows;
    alloc.assignedPoolId = allocation.assignedPoolId;
    alloc.token = allocation.token;
    for (uint256 j; j < nbOfFlows; ) {
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
All calls to that will revert with `array out-of-bounds access`.

### Root Cause

Allocation arrays are not initialized in `_computeSplitArrays`, https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1121.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

N/A

### Impact

Splitting TVS can never be done.

### PoC

Apply the following diff before running the test (fixes a couple of issues that were reported separately):
```diff
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+   newAmounts = new uint256[](length);
    for (uint256 i; i < length; ) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
+       i++;
    }
}
```

```solidity
function test_UninitializedArraysSplit() public {
    address user1 = makeAddr("user1");

    uint256 rewardProjectId = vesting.rewardProjectCount();
    vm.prank(projectCreator);
    vesting.launchRewardProject(
        address(token),
        address(usdt),
        block.timestamp + 10 days,
        30 days
    );

    address[] memory kols = new address[](1);
    kols[0] = user1;
    uint256[] memory kolRewards = new uint256[](1);
    kolRewards[0] = 100e18;
    vm.prank(projectCreator);
    vesting.setTVSAllocation(
        rewardProjectId,
        100e18,
        30 days,
        kols,
        kolRewards
    );

    uint256 user1Nft = nft.totalSupply() + 1;
    vm.prank(user1);
    vesting.claimRewardTVS(rewardProjectId);

    uint256[] memory percentages = new uint256[](2);
    percentages[0] = 5000;
    percentages[1] = 5000;

    vm.prank(user1);
    vm.expectRevert(); // array out-of-bounds access
    vesting.splitTVS(rewardProjectId, percentages, user1Nft);
}
```

### Mitigation

Consider initializing all needed arrays before setting the values in `_computeSplitArrays`:
```solidity
alloc.amounts = new uint256[](nbOfFlows);
alloc.vestingPeriods = new uint256[](nbOfFlows);
alloc.vestingStartTimes = new uint256[](nbOfFlows);
alloc.claimedSeconds = new uint256[](nbOfFlows);
alloc.claimedFlows = new bool[](nbOfFlows);
```
  