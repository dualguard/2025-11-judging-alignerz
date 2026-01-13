# [000108] `_computeSplitArrays` does not initialize `allocation` arrays, making the function always revert
  
  ### Summary

Function [`_computeSplitArrays`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113) does not initialize it's own arrays in the struct, thus it will always revert with error `array out-of-bounds` when called in [`splitTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054).
```solidity
    function splitTVS(
        uint256 projectId,
        uint256[] calldata percentages,
        uint256 splitNftId
    ) external returns (uint256, uint256[] memory) {
        ...
@>           Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
        ...
```

### Root Cause

Function [_computeSplitArrays](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113) does not initialize in the struct `Allocation memory alloc`.

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

1. User calls `splitTVS`.
2. Function always reverts.

### Impact

A core functionality of the protocol (splitting TVS) is fully broken with no ways of calling the function since it will always revert.

### PoC

Put this in the AlignerzVestingProtocolTest.t.sol file:
```solidity
    function test_PoC_TVSSplittingDoesNotWork() public {
        vm.startPrank(projectCreator);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 1);

        address[] memory kols = new address[](10);
        for (uint256 i = 0; i < 10; i++) {
            kols[i] = address(uint160(i + 1));
        }

        uint256[] memory amounts = new uint256[](10);
        for (uint256 i = 0; i < 10; i++) {
            amounts[i] = 1_000e18;
        }

        vesting.setTVSAllocation(0, 10_000e18, 30 days, kols, amounts);
        vm.stopPrank();

        for (uint256 i = 0; i < 10; i++) {
            vm.prank(kols[i]);
            vesting.claimRewardTVS(0);
        }

        vm.warp(block.timestamp + 29 days);

        vm.startPrank(kols[0]);

        nft.setApprovalForAll(address(vesting), true);
        vesting.claimTokens(0, 1);

        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5_000;
        percentages[1] = 5_000;

        vm.expectRevert();
        vesting.splitTVS(0, percentages, 1);

        vm.stopPrank();
    }
```

### Mitigation

Initialize the `allocation` arrays:
```diff
    function _computeSplitArrays(Allocation storage allocation, uint256 percentage, uint256 nbOfFlows)
        internal
        view
        returns (Allocation memory alloc)
    {
+       uint256[] memory baseAmounts = allocation.amounts;
+       uint256[] memory baseVestings = allocation.vestingPeriods;
+       uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
+       uint256[] memory baseClaimed = allocation.claimedSeconds;
+       bool[] memory baseClaimedFlows = allocation.claimedFlows;

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
  