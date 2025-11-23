# [000031] Incorrect split logic causes permanent loss of vested amounts
  
  ### Summary

The `splitTVS` function is intended to split a single TVS (represented by `splitNftId`) into multiple TVSs according to a list of percentages, preserving the total vested amount (minus any configured fee):

```solidity
function splitTVS(
    uint256 projectId,
    uint256[] calldata percentages,
    uint256 splitNftId
) external returns (uint256, uint256[] memory) {
    ...
    bool isBiddingProject = NFTBelongsToBiddingProject[splitNftId];
    (Allocation storage allocation, IERC20 token) = isBiddingProject
        ? (
            biddingProjects[projectId].allocations[splitNftId],
            biddingProjects[projectId].token
        )
        : (
            rewardProjects[projectId].allocations[splitNftId],
            rewardProjects[projectId].token
        );

    uint256[] memory amounts = allocation.amounts;
    uint256 nbOfFlows = allocation.amounts.length;
    (uint256 feeAmount, uint256[] memory newAmounts) =
        calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);

    allocation.amounts = newAmounts;                  // <--- storage mutation

    uint256 nbOfTVS = percentages.length;
    uint256[] memory newNftIds = new uint256[](nbOfTVS - 1);
    Allocation[] memory allAlloc = new Allocation[](nbOfTVS);

    uint256 sumOfPercentages;
    for (uint256 i; i < nbOfTVS; ) {
        uint256 percentage = percentages[i];
        sumOfPercentages += percentage;

        uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);
        if (i != 0) newNftIds[i - 1] = nftId;

        Allocation memory alloc = _computeSplitArrays(
            allocation,              // <--- uses the mutated storage allocation
            percentage,
            nbOfFlows
        );

        NFTBelongsToBiddingProject[nftId] = isBiddingProject ? true : false;
        Allocation storage newAlloc = isBiddingProject
            ? biddingProjects[projectId].allocations[nftId]
            : rewardProjects[projectId].allocations[nftId];

        _assignAllocation(newAlloc, alloc);
        allocationOf[nftId] = newAlloc;
        ...
        unchecked { ++i; }
    }
    require(
        sumOfPercentages == BASIS_POINT,
        Percentages_Do_Not_Add_Up_To_One_Hundred()
    );
    return (splitNftId, newNftIds);
}
```

The code uses the same `allocation` storage reference as both:

1. The **base allocation** being split, and
2. The **target allocation** for the first NFT (`splitNftId`) in the loop.

Inside `_computeSplitArrays`, the allocation is treated as a base snapshot:

```solidity
function _computeSplitArrays(
    Allocation memory allocation,
    uint256 percentage,
    uint256 nbOfFlows
) internal view returns (Allocation memory alloc) {
    uint256[] memory baseAmounts = allocation.amounts;
    ...
    alloc.amounts = new uint256[](nbOfFlows);
    ...
    for (uint256 j; j < nbOfFlows; ) {
        alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
        ...
        unchecked { ++j; }
    }
}
```

However, `allocation` passed into `_computeSplitArrays` is **not** a fixed snapshot – it is the same storage-backed `allocation` that is being overwritten in each iteration:

* On the first iteration, the code computes 50% (for example) of the original amount and writes that back to `allocation` for `splitNftId`.
* On the second iteration, `_computeSplitArrays` sees a **already-reduced** `allocation.amounts` and applies the second percentage on this reduced amount, effectively computing a percentage of a percentage.

This means:

* The first NFT gets `percentage[0] * originalAmount / 10000`.
* The second NFT gets `percentage[1] * (firstResult) / 10000` instead of `percentage[1] * originalAmount / 10000`.

As a result, the sum of all split TVSs is strictly less than the original TVS. The difference remains in the contract’s token balance and is never assigned to any allocation, so it can never be claimed.

### Root Cause

The use of in-place mutation of the base allocation, https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1088.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

N/A

### Impact

Splitting a vesting schedule (`TVS`) can permanently destroy a portion of the user’s vested balance: the sum of all post-split TVSs is strictly less than the original allocation, and the “missing” tokens remain stuck in the vesting contract and can never be claimed.

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

```diff
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
+   alloc.amounts = new uint256[](nbOfFlows);
+   alloc.vestingPeriods = new uint256[](nbOfFlows);
+   alloc.vestingStartTimes = new uint256[](nbOfFlows);
+   alloc.claimedSeconds = new uint256[](nbOfFlows);
+   alloc.claimedFlows = new bool[](nbOfFlows);
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

```solidity
function test_LossOfVestedAmount() public {
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
    vesting.splitTVS(rewardProjectId, percentages, user1Nft);

    vm.warp(block.timestamp + 50 days);

    uint256 balanceBefore = token.balanceOf(user1);

    vm.prank(user1);
    vesting.claimTokens(rewardProjectId, user1Nft);

    vm.prank(user1);
    vesting.claimTokens(rewardProjectId, user1Nft + 1);

    uint256 balanceAfter = token.balanceOf(user1);

    assertEq(
        balanceAfter - balanceBefore,
        75e18,
        "Wrong token amount claimed"
    );
    assertEq(
        token.balanceOf(address(vesting)),
        25e18,
        "Vested amount lost during split"
    );
}
```

### Mitigation

Consider treating the TVS to be split as an **immutable base snapshot** when computing split allocations. For example:

  ```solidity
  Allocation memory base = allocationOf[splitNftId];

  // Apply split fee once on the base amounts
  (uint256 feeAmount, uint256[] memory newBaseAmounts) =
      calculateFeeAndNewAmountForOneTVS(splitFeeRate, base.amounts, nbOfFlows);
  base.amounts = newBaseAmounts;

  // Optionally update the original storage allocation after snapshot
  allocation.amounts = newBaseAmounts;

  for (uint256 i; i < nbOfTVS; ) {
      uint256 percentage = percentages[i];
      uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);

      Allocation memory alloc = _computeSplitArrays(
          base,            // <- always use the same base snapshot
          percentage,
          nbOfFlows
      );

      Allocation storage newAlloc = isBiddingProject
          ? biddingProjects[projectId].allocations[nftId]
          : rewardProjects[projectId].allocations[nftId];

      _assignAllocation(newAlloc, alloc);
      allocationOf[nftId] = newAlloc;

      unchecked { ++i; }
  }
  ```
  