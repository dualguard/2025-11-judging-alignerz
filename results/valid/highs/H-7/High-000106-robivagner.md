# [000106] `calculateFeeAndNewAmountForOneTVS` does not increase `i` in for loop, always reverting
  
  ### Summary

Function [`calculateFeeAndNewAmountForOneTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169) does not increase index `i` in for loop, making it to always revert when called in [`mergeTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002) with error `out of gas` (infinite loop):
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
    function mergeTVS(uint256 projectId, uint256 mergedNftId, uint256[] calldata projectIds, uint256[] calldata nftIds)
        external
        returns (uint256)
    {
        ...
        (uint256 feeAmount, uint256[] memory newAmounts) =
@>            calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
        ...
```

### Root Cause

Function [calculateFeeAndNewAmountForOneTVS](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169) does not increase index i in for loop.

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

1. `mergeTVS` is called.
2. Function always reverts.

### Impact

A core functionality of the protocol is fully broken with no ways of calling the function since it will always revert.

### PoC

```solidity
    function test_PoC_TVSMergingDoesNotWork() public {
        vm.startPrank(projectCreator);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 1);
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

        vm.startPrank(projectCreator);

        address[] memory attacker = new address[](1);
        attacker[0] = kols[0];
        uint256[] memory amount = new uint256[](1);
        amount[0] = 1_000e18;
        vesting.setTVSAllocation(1, 1_000e18, 30 days, attacker, amount);
        vm.stopPrank();

        vm.prank(kols[0]);
        vesting.claimRewardTVS(1);

        vm.warp(block.timestamp + 29 days);

        vm.startPrank(kols[0]);

        nft.setApprovalForAll(address(vesting), true);
        vesting.claimTokens(0, 1);
        vesting.claimTokens(1, 11);

        uint256[] memory projectIds = new uint256[](1);
        uint256[] memory nftIds = new uint256[](1);
        projectIds[0] = 1;
        nftIds[0] = 11;

        vm.expectRevert();
        vesting.mergeTVS(0, 1, projectIds, nftIds);
        vm.stopPrank();
    }
```

### Mitigation

```diff
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
+            unchecked {
+               ++i;
+            }
        }
```
  