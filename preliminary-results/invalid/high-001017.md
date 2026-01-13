# [001017] Unbounded for loop in `claimTokens` may result in `claimTokens()` DoS
  
  ### Summary

There's no limitation on how many times a TVS can be split and merged. With each merge, the number of flows a TVS has is increased. 

The unbounded for loop from the `claimTokens()` can exceed the block gas limit resulting in a tokens claiming DOS for that TVS. 

### Root Cause

The new TVS created by the merge has a number of flows [equal](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1038-L1046) to the sum of the original TVSs' flows.
There's no limit of how many flows a TVS can have. 

`claimTokens()` [loops](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L953-L968) over all flows to claim the vested tokens from each of them:


```solidity
    function claimTokens(uint256 projectId, uint256 nftId) external {
//...
        uint256 nbOfFlows = allocation.vestingPeriods.length; // @audit get the total flow number
//...
        uint256 flowsClaimed;
        for (uint256 i; i < nbOfFlows; i++) {     // @audit unbounded loop execution can exceed the block gas limit
            if (allocation.claimedFlows[i]) {
                flowsClaimed++;
                continue;
            }
            (uint256 claimableAmount, uint256 claimableSeconds) = getClaimableAmountAndSeconds(allocation, i);
//...
        }
//...
```


Due to unbounded for loop, the `claimTokens()` execution may require more gas than the max gas block limit. 
In these cases the tx reverts and users can't claim their tokens. 

### Other issues with same root cause - unbounded loop
The following functions are are vulnerable to DoS due to unbounded loop:
- [getTotalUnclaimedAmounts](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L129) loop over all NFT minted; for each NFT it calls [getUnclaimedAmounts](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L147) which loops over all flows, making things worse. This function, `getTotalUnclaimedAmounts` is [called](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L209) to set the  dividends details. 
- [_setDividends](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L216) loops over all NFT id to calculate and set the dividend for each holder.
- 
Dividend details may not be set due to above issues. 
Since one admin function loops over all flows from all TVS, the likelihood of exceeding block limit gas is high, thus the High severity. 

### Internal Pre-conditions

1. There must be a TVS with a high enough number of flows. 

### External Pre-conditions

None

### Attack Path

The number of flows grows exponentially with each split and merge. 
Let's consider Alice has 2 TVS, both with 2 flows only: 
1. Alice split both TVS in 3 other TVSs: now each resulting TVSs (2 *3 = 6 in total) have 2 flows each. 
2. After a while Alice merge all her 6 TVS into one: the resulting TVS has 6 * 2 = 12 flows. 

Starting from really low numbers (2 TVS with 2 flows each), only after one split& merge cycle Alice got a TVS with 12 flows. 

Since holders can split and merge their TVS as they desire the number of flows may grow high enough for `claimTokens()` to exceed the block gas limit.


### Impact

- User can't claim the tokens from that TVS. 
- Admin can't set the dividend details. 

### PoC

_No response_

### Mitigation

Consider implementing one of the following suggestions:
1. limit the number of flows a TVS can have after merging OR
2. allow holders to claim vested tokens for an interval of flows. 
Instead of looping over all flows, allow users to pass a `startIndexFlow` and `endIndexFlow` argument and process only the flows from this interval.
  