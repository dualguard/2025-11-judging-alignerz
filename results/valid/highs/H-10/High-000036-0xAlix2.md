# [000036] Merge/split fees are based on the whole vesting amount, causing claim DoS
  
  ### Summary

When merging/splitting TVS, some fees are applied to the merged amounts. These fees are cut from the amounts and sent to the treasury.
For merging TVS, fees are cut first from the merged NFT:
```solidity
(uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
mergedTVS.amounts = newAmounts;
```
In addition to the NFTs being merged, where fees are cut from each in the `_merge` function:
```solidity
for (uint256 j = 0; j < nbOfFlowsTVSToMerge; j++) {
    uint256 fee = calculateFeeAmount(mergeFeeRate, TVSToMerge.amounts[j]);
    mergedTVS.amounts.push(TVSToMerge.amounts[j] - fee);
    // ... snip ...
}
```

On the other hand, these amounts are vested amounts, and when a user claims, the amount doesn't get subtracted, but the claimed seconds are saved to check how much of the vesting was claimed. However, this is not accounted for here, where fees are applied to the whole vested amount, regardless of whether part of it was claimed or not.

This would charge fees more than what's expected/available, draining the contract, and blocking future claims from being fulfilled.

### Root Cause

Merging/splitting fees are based on the whole vested amounts rather than the unclaimed amounts.
* https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1014
* https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1021
* https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1070

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

N/A

### Impact

Users will be blocked from claiming their vested tokens, as the contract won't hold enough tokens.

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
function test_WrongFeesCut() public {
    address user1 = makeAddr("user1");
    address user2 = makeAddr("user2");

    vm.prank(projectCreator);
    vesting.setMergeFeeRate(200);

    uint256 rewardProjectId = vesting.rewardProjectCount();
    vm.prank(projectCreator);
    vesting.launchRewardProject(
        address(token),
        address(usdt),
        block.timestamp + 10 days,
        30 days
    );

    address[] memory kols = new address[](2);
    kols[0] = user1;
    kols[1] = user2;
    uint256[] memory kolRewards = new uint256[](2);
    kolRewards[0] = 100e18;
    kolRewards[1] = 200e18;
    vm.prank(projectCreator);
    vesting.setTVSAllocation(
        rewardProjectId,
        300e18,
        30 days,
        kols,
        kolRewards
    );

    uint256 user1Nft = 1;
    vm.prank(user1);
    vesting.claimRewardTVS(rewardProjectId);

    uint256 user2Nft = 2;
    vm.prank(user2);
    vesting.claimRewardTVS(rewardProjectId);

    vm.warp(block.timestamp + 20 days);

    vm.prank(user1);
    vesting.claimTokens(rewardProjectId, user1Nft);

    vm.prank(user2);
    nft.transferFrom(user2, user1, user2Nft);

    uint256[] memory projectIds = new uint256[](1);
    projectIds[0] = rewardProjectId;
    uint256[] memory nftIds = new uint256[](1);
    nftIds[0] = user2Nft;
    vm.prank(user1);
    vesting.mergeTVS(rewardProjectId, user1Nft, projectIds, nftIds);

    vm.warp(block.timestamp + 20 days);

    vm.prank(user1);
    vm.expectRevert(); // ERC20InsufficientBalance
    vesting.claimTokens(rewardProjectId, user1Nft);
}
```

### Mitigation

Consider applying the fee to the unclaimed amount rather than the entire vesting amount, which may include some previously claimed part.
  