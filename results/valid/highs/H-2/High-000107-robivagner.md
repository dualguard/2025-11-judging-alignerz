# [000107] `calculateFeeAndNewAmountForOneTVS` does not initialize array correctly, allowing it to always revert
  
  ### Summary

Function [`calculateFeeAndNewAmountForOneTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169) does not initialize `newAmounts` array, thus it will always revert with error `array out-of-bounds` when called in [`splitTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054).
```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
        public
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```
```solidity
    function splitTVS(
        uint256 projectId,
        uint256[] calldata percentages,
        uint256 splitNftId
    ) external returns (uint256, uint256[] memory) {
        ...
@>        (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);
        ...
```

### Root Cause

Function [calculateFeeAndNewAmountForOneTVS](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169) does not initialize array `uint256[] memory newAmounts`.

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

1. `splitTVS` is called.
2. Function always reverts.

### Impact

Core functionality of the protocol is fully broken and will never be able to be used.

### PoC

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

Initialize the array:
```diff
    function calculateFeeAndNewAmountForOneTVS_correct(uint256 feeRate, uint256[] memory amounts, uint256 length)
        public
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
+       newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
  