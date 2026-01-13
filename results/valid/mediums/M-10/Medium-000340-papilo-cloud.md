# [000340] Unlimited flow accumulation will cause exponential gas cost increases and render TVS operations unusable
  
  ### Summary

The `_merge()` function always appends flows using `push()` instead of combining compatible flows, which will cause linear growth in flow count with each merge operation as users who repeatedly merge TVSs will accumulate N×M flows (where N = number of merges, M = flows per TVS), making all future operations expensive in gas costs and potentially hitting block gas limits.

### Root Cause

In [`AlignerzVesting.sol:_merge()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1028), flows are appended to the destination TVS without checking if they can be combined:

```solidity
function _merge(...) internal returns (uint256 feeAmount) {
    // ... 
    
    uint256 nbOfFlowsTVSToMerge = TVSToMerge.amounts.length;
    
    // @audit-issue: Always uses push(), never combines compatible flows
    for (uint256 j = 0; j < nbOfFlowsTVSToMerge; j++) {
        uint256 fee = calculateFeeAmount(mergeFeeRate, TVSToMerge.amounts[j]);
        
        // These push() calls ALWAYS create new flows
        mergedTVS.amounts.push(TVSToMerge.amounts[j] - fee);
        mergedTVS.vestingPeriods.push(TVSToMerge.vestingPeriods[j]);
        mergedTVS.vestingStartTimes.push(TVSToMerge.vestingStartTimes[j]);
        mergedTVS.claimedSeconds.push(TVSToMerge.claimedSeconds[j]);
        mergedTVS.claimedFlows.push(TVSToMerge.claimedFlows[j]);
        
        feeAmount += fee;
    }
    // ...
}
```
Even when flows have identical vesting parameters like `vestingPeriod` and `vestingStartTime`, they are not combined, resulting in unnecessary duplication.

### Internal Pre-conditions

1. User must own multiple TVS NFTs with flows that have matching vesting parameters
2. User must perform multiple merge operations over time
3. Each TVS must have at least 1 flow

### External Pre-conditions

_No response_

### Attack Path

1. User starts with TVS A containing 1 flow: `[1000 tokens, 12 months vesting, start: Jan 1]`
2. User merges TVS B (also 1 flow with identical params: `[2000 tokens, 12 months vesting, start: Jan 1]`)
3. Instead of combining into 1 flow of 3000 tokens, the result is 2 separate flows: `[1000 tokens], [2000 tokens]`
4. User later merges TVS C (1 flow, same params: `[3000 tokens, 12 months vesting, start: Jan 1]`)
5. Result is now 3 flows: `[1000 tokens], [2000 tokens], [3000 tokens]`
6. After 10 merges, the user has 10 separate flows, all with identical vesting parameters
7. Each subsequent operation like claim and split must iterate through all 10 flows
8. Gas costs for operations increase linearly
9. After 50 merges, gas costs become prohibitively expensive (potential DoS)
10. Eventually, the user's TVS becomes practically unusable due to gas costs exceeding block limits

### Impact

Users experience exponentially increasing gas costs for all TVS operations. In a worst-case scenario where a user performs 20 merges (starting with 2 flows per TVS), they would accumulate 40 flows. Every claim operation would need to iterate through 40 flows, consuming 40x more gas than necessary. This renders the TVS economically unusable, as gas costs could exceed the value of the tokens being claimed. Additionally, this creates a permanent bloat issue where TVS data structures grow unbounded, eventually hitting EVM storage and execution limits.

### PoC

_No response_

### Mitigation

Implement flow combination logic that merges compatible flows:

```solidity
// For this ocde, I removed the fees charged for merging
function _merge(...) internal {
    // ...

    uint256 nbOfFlowsTVSToMerge = TVSToMerge.amounts.length;
    
    for (uint256 j = 0; j < nbOfFlowsTVSToMerge; j++) {
        // look for a match flow i
        bool foundMatch = false;
        
        for (uint256 k = 0; k < mergedTVS.amounts.length; k++) {
            // Check if flows can be combined (same vesting params)
            if (mergedTVS.vestingPeriods[k] == TVSToMerge.vestingPeriods[j] &&
                mergedTVS.vestingStartTimes[k] == TVSToMerge.vestingStartTimes[j] &&
                !mergedTVS.claimedFlows[k] && !TVSToMerge.claimedFlows[j]) {
                
                // Calculate claimed amounts for proper recalculation
                uint256 claimed1 = (mergedTVS.amounts[k] * mergedTVS.claimedSeconds[k]) / mergedTVS.vestingPeriods[k];
                uint256 claimed2 = (TVSToMerge.amounts[j] * TVSToMerge.claimedSeconds[j]) / TVSToMerge.vestingPeriods[j];
                
                uint256 newTotalAmount = mergedTVS.amounts[k] + TVSToMerge.amounts[j];
                uint256 totalClaimed = claimed1 + claimed2;
                
                // Combine the amounts
                mergedTVS.amounts[k] = newTotalAmount;
                
                // Recalculate claimedSeconds based on combined claimed amounts
                if (newTotalAmount > 0) {
                    mergedTVS.claimedSeconds[k] = (totalClaimed * mergedTVS.vestingPeriods[k]) / newTotalAmount;
                }
                
                foundMatch = true;
                break;
            }
        }
        
        // If no compatible flow found, append as new flow
        if (!foundMatch) {
            mergedTVS.amounts.push(TVSToMerge.amounts[j]);
            mergedTVS.vestingPeriods.push(TVSToMerge.vestingPeriods[j]);
            mergedTVS.vestingStartTimes.push(TVSToMerge.vestingStartTimes[j]);
            mergedTVS.claimedSeconds.push(TVSToMerge.claimedSeconds[j]);
            mergedTVS.claimedFlows.push(TVSToMerge.claimedFlows[j]);
        }
    }
    
    nftContract.burn(nftId);
}
```

Implementing a maximum flow count limit to can prevent unbounded growth:
  