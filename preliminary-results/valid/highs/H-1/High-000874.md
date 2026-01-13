# [000874] Storage-referenced allocation variable alters the values for future iterations in `splitTVS()`
  
  ### Summary

`allocation` variable is initialized as a storage  in `splitTVS()`
```solidity
        (Allocation storage allocation, IERC20 token) = isBiddingProject ?
```
It is later used in the core for loop that creates the new NFTs and assigns allocation to them.

On the first iteration `i == 0`, `_computeSplitArrays` uses `allocation.amounts` and writes a first split:
```solidity
            Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
```
v
```solidity
    function _computeSplitArrays(
@>      Allocation storage allocation,
        uint256 percentage,
        uint256 nbOfFlows
    )
        internal
        view
        returns (
            Allocation memory alloc
        )
    {
        uint256[] memory baseAmounts = allocation.amounts;
        uint256[] memory baseVestings = allocation.vestingPeriods;
        uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
        uint256[] memory baseClaimed = allocation.claimedSeconds;
        bool[] memory baseClaimedFlows = allocation.claimedFlows;
        alloc.assignedPoolId = allocation.assignedPoolId;
        alloc.token = allocation.token;
        for (uint256 j; j < nbOfFlows;) {
            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
```
`_assignAllocation()` then writes that split back into the same `allocation` storage variable for `splitNftId`:
```solidity
            Allocation storage newAlloc = isBiddingProject ? biddingProjects[projectId].allocations[nftId] : rewardProjects[projectId].allocations[nftId];
            _assignAllocation(newAlloc, alloc);
```
In the code above, `nftId` is the same as the `splitNftId` used for the initial `allocation` variable definition - as we are on the first iteration (`uint256 nftId = i == 0 ? splitNftId :`).
```solidity
    function _assignAllocation(
        Allocation storage allocation, // @audit same storage as the original allocation used for all the other NFTs 
        Allocation memory alloc
    ) internal {
        allocation.amounts = alloc.amounts;
```

Because of this, on `i == 1`, the `_computeSplitArrays()` function sees `allocation.amounts` already reduced by the first split ( ie. it returns the allocation for the `splitNftId` and not the total allocation for all split NFTs ).

### Root Cause

Usage of the same storage variable for the calculation of split allocations as the original allocation amount, which is updated on first iteration. 

### Internal Pre-conditions

-

### External Pre-conditions

-

### Attack Path

-

### Impact

Tokens that are unallocated will be stuck in the contract, user losses their allocation for no reason.

How big is the loss is inversely proportional to the allocation of the first ( original ) `splitNftId` - the smaller the allocation, the larger the loss. 

### PoC

-

### Mitigation

Use memory variables for the `splitNftId` on first iteration and then explicitly write it to storage, or use a memory variable for the original allocation used to calculate split allocations. 
  