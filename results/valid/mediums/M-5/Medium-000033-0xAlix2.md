# [000033] Underflow in claimable seconds can block vesting claims after merge
  
  ### Summary

The function `getClaimableAmountAndSeconds` computes `secondsPassed` as:

```solidity
function getClaimableAmountAndSeconds(
    Allocation memory allocation,
    uint256 flowIndex
) public view returns (uint256 claimableAmount, uint256 claimableSeconds) {
    uint256 secondsPassed;
    uint256 claimedSeconds = allocation.claimedSeconds[flowIndex];
    uint256 vestingPeriod = allocation.vestingPeriods[flowIndex];
    uint256 vestingStartTime = allocation.vestingStartTimes[flowIndex];
    uint256 amount = allocation.amounts[flowIndex];

    if (block.timestamp > vestingPeriod + vestingStartTime) {
        secondsPassed = vestingPeriod;
    } else {
        secondsPassed = block.timestamp - vestingStartTime;
    }

    claimableSeconds = secondsPassed - claimedSeconds;
    claimableAmount = (amount * claimableSeconds) / vestingPeriod;
    require(claimableAmount > 0, No_Claimable_Tokens());
}
```

If `block.timestamp < vestingStartTime`, the expression:

```solidity
secondsPassed = block.timestamp - vestingStartTime;
```

underflows, causing a revert. Because `claimTokens` loops over all flows and calls this function for each one, a single flow with a `vestingStartTime` in the future will cause the entire `claimTokens` call to revert.

This becomes reachable in a realistic way when the user merges TVSs from different projects with different `startTime` values. After merging, the resulting TVS can contain flows where some vesting has already started and others have a `vestingStartTime` in the future. 

Calling `claimTokens` during this “mixed” period will revert for the future-dated flow and prevent claiming for all flows.


### Root Cause

`getClaimableAmountAndSeconds` doesn't handle the case where `vestingStartTime` is in the future, https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L989.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

N/A

### Impact

Users holding merged TVSs that contain flows with different `vestingStartTime` values may be temporarily unable to claim any of their vested tokens.

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
function test_UnderflowClaimAmount() public {
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

    uint256 rewardProjectId2 = vesting.rewardProjectCount();
    vm.prank(projectCreator);
    vesting.launchRewardProject(
        address(token),
        address(usdt),
        block.timestamp,
        30 days
    );

    vm.prank(projectCreator);
    vesting.setTVSAllocation(
        rewardProjectId2,
        100e18,
        30 days,
        kols,
        kolRewards
    );

    uint256 user1Nft2 = nft.totalSupply() + 1;
    vm.prank(user1);
    vesting.claimRewardTVS(rewardProjectId2);

    uint256[] memory projectIds = new uint256[](1);
    projectIds[0] = rewardProjectId2;
    uint256[] memory nftIds = new uint256[](1);
    nftIds[0] = user1Nft2;
    vm.prank(user1);
    vesting.mergeTVS(rewardProjectId, user1Nft, projectIds, nftIds);

    vm.warp(block.timestamp + 5 days);

    vm.prank(user1);
    vm.expectRevert(); // arithmetic underflow or overflow
    vesting.claimTokens(rewardProjectId, user1Nft);
}
```

### Mitigation

Consider safely handling flows whose vesting has not started yet by short-circuiting before subtracting:
```solidity
if (block.timestamp <= vestingStartTime) return (0, 0);
```
  