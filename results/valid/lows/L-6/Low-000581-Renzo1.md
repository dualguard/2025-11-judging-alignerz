# [000581] Array Length Mismatch in Loop Iteration Causes Out-of-Bounds Access and Function Failure in `distributeRewardTVS()`
  
  
## Summary

A mismatch between the loop iteration count and the array being accessed in `distributeRewardTVS()` causes out-of-bounds access or incomplete distribution for KOLs and the protocol, as the owner calls the function with an array length that doesn't match the stored addresses array length.

## Root Cause

In [`AlignerzVesting.sol:525-535`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L525-L535), the loop uses `rewardProject.kolTVSAddresses.length` for the iteration count but accesses `kol[i]` from the function parameter. This creates a mismatch: the loop bounds come from the stored array while the accessed array is the input parameter.
```js
    function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
        uint256 len = rewardProject.kolTVSAddresses.length; // <--@audit
        for (uint256 i; i < len;) {
            _claimRewardTVS(rewardProjectId, kol[i]);
            unchecked {
                ++i;
            }
        }
    }
```
## Internal Pre-conditions

1. Owner needs to call `setTVSAllocation()` to populate `rewardProject.kolTVSAddresses` with at least one address
2. `block.timestamp` needs to be greater than `rewardProject.claimDeadline` for the function to execute
3. Owner needs to call `distributeRewardTVS()` with a `kol` array where `kol.length` differs from `rewardProject.kolTVSAddresses.length`

## External Pre-conditions

None

## Attack Path

1. Owner calls `setTVSAllocation()` to set TVS allocations for multiple KOLs, populating `rewardProject.kolTVSAddresses` with N addresses
2. The claim deadline passes (`block.timestamp > rewardProject.claimDeadline`)
3. Owner calls `distributeRewardTVS()` with a `kol` array of length M, where M ≠ N
4. If M < N: the loop iterates N times but accesses `kol[0]` through `kol[M-1]`, causing an out-of-bounds access at `kol[N-1]` and reverting
5. If M > N: the loop iterates only N times, ignoring addresses `kol[N]` through `kol[M-1]`, leaving some KOLs without distribution

## Impact

- If `kol.length < kolTVSAddresses.length`: the transaction reverts due to out-of-bounds access, preventing distribution
- If `kol.length > kolTVSAddresses.length`: some KOLs in the `kol` array are not processed, leaving their rewards undistributed
- The function is effectively broken and unusable in its current form

## Proof of Concept

N/A


## Mitigation

Use the parameter array length for iteration:

```solidity
function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
    uint256 len = kol.length; // Use the parameter array length
    for (uint256 i; i < len;) {
        _claimRewardTVS(rewardProjectId, kol[i]);
        unchecked {
            ++i;
        }
    }
}
```


  