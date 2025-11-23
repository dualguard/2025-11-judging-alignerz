# [000173] User can make NFT with many flows to DoS the dividend distribution
  
  ### Summary

User is able to make an NFT with as many flows as possible to DoS the distribution. This is possible due to the fact that a user is able to merge and split NFTs at his will, which as can be seen will add new flows. Consider the following exmaple:
1. User calls the [`AlignerzVesting::splitTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054) function which produces 2 NFTs with the same number of flows
2. User calls the [`AlignerzVesting::mergeTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002) function, which combines the NFTs, pushing the number of flows of the burned NFT to the already existing one as can be seen here:
```solidity
       for (uint256 j = 0; j < nbOfFlowsTVSToMerge; j++) {
            uint256 fee = calculateFeeAmount(
                mergeFeeRate,
                TVSToMerge.amounts[j]
            );
            mergedTVS.amounts.push(TVSToMerge.amounts[j] - fee);
            mergedTVS.vestingPeriods.push(TVSToMerge.vestingPeriods[j]);
            mergedTVS.vestingStartTimes.push(TVSToMerge.vestingStartTimes[j]);
            mergedTVS.claimedSeconds.push(TVSToMerge.claimedSeconds[j]);
            mergedTVS.claimedFlows.push(TVSToMerge.claimedFlows[j]);
            feeAmount += fee;
        }
        nftContract.burn(nftId);
    }
```

This whole scenario will lead to DoS of the `DividendDistribution` contract, because the `for` loop in the `getUnclaimedAmounts` function is strongly dependant on the  number of flows of the corresponding NFTs as can be seen in the following line of code:
```solidity
  function getUnclaimedAmounts(
        uint256 nftId
    ) public returns (uint256 amount) {
        if (address(token) == address(vesting.allocationOf(nftId).token))
            return 0;
        uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
        uint256[] memory claimedSeconds = vesting
            .allocationOf(nftId)
            .claimedSeconds;
        uint256[] memory vestingPeriods = vesting
            .allocationOf(nftId)
            .vestingPeriods;
        bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
        uint256 len = vesting.allocationOf(nftId).amounts.length;
@>        for (uint i; i < len; ) {          <@
```

### Root Cause

Root cause of the issue is that a user can have as many flows attached to 1 NFT as he wishes, thanks to the way merge and split functions work

### Internal Pre-conditions

User having and NFT minted by the system form either reward project or biding project

### External Pre-conditions

None

### Attack Path

1. User has an NFT and starts calling split and merge on it so that it's flows become more
2. The `getUnclaimedAmounts` function exceeds the gas limit of the corresponding chain because it has to iterate through too many flows.

### Impact

Permanent DoS for the `DividendDistributor` contract 

### PoC

Not needed

### Mitigation

Limit the max number of flows on an NFTs
  