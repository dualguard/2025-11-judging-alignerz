# [000687] Mismatch between loop bound and input array length causes "Index Out of Bounds" revert and Dos in distribution functions
  
  ### Summary

The functions `distributeRewardTVS` and `distributeStablecoinAllocation` use the length of the *storage* array (`rewardProject.kolTVSAddresses`) to determine the number of iterations, but they access elements from the *input* array (`kol`). This mismatch causes the transaction to revert with an "Index Out of Bounds" error whenever the input array is smaller than the total number of remaining users. This effectively breaks the batching functionality, leading to a DoS if the total number of users exceeds the block gas limit, as the owner cannot process them in smaller chunks.

### Root Cause

In `AlignerzVesting.sol`, the loop iteration count (`len`) is derived from the contract's storage state, but the index `i` is used to access the function's calldata argument.
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L522-L551
```solidity
// AlignerzVesting.sol

function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
    // ...
    // ROOT CAUSE: len is set to the TOTAL remaining count in storage
    uint256 len = rewardProject.kolTVSAddresses.length; 
    
    // The loop runs 'len' times
    for (uint256 i; i < len;) {
        // VULNERABILITY: If kol.length < len, accessing kol[i] will REVERT here
        _claimRewardTVS(rewardProjectId, kol[i]); 
        unchecked {
            ++i;
        }
    }
}
```

The logic assumes that the input array `kol` always contains *all* remaining users. If the caller attempts to pass a partial list (batching), `i` will eventually exceed `kol.length - 1`.

### Internal Pre-conditions

1.  The `claimDeadline` has passed.
2.  The number of remaining unclaimed users in storage (`rewardProject.kolTVSAddresses.length`) is greater than `0` (e.g., 100 users).

### External Pre-conditions

_No response_

### Attack Path

1.  Assume there are 100 users remaining in the project who haven't claimed their rewards.
2.  The Owner attempts to process these users in batches to avoid high gas costs.
3.  The Owner calls `distributeRewardTVS` with an array `kol` containing the first 10 addresses.
4.  The function sets `len = 100` (from storage).
5.  The loop runs successfully for indices 0 to 9.
6.  On the next iteration, `i = 10`. The loop condition `10 < 100` is true.
7.  The code attempts to access `kol[10]`.
8.  Since `kol` only has a length of 10, this access is out of bounds.
9.  The transaction reverts (Panic code 0x32).

### Impact

*   **Broken Functionality:** The ability to distribute rewards in batches is completely non-functional.
*   **DoS:** If the total number of remaining users is large enough that processing all of them in one transaction exceeds the Block Gas Limit, the Owner cannot distribute the rewards at all. They cannot send a full list (Out of Gas) and they cannot send a partial list (Index Out of Bounds revert).

### PoC

_No response_

### Mitigation

Update the loop bound to use the length of the input array `kol` instead of the storage array.

```diff
    function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
        
-       uint256 len = rewardProject.kolTVSAddresses.length;
+       uint256 len = kol.length;

        for (uint256 i; i < len;) {
            _claimRewardTVS(rewardProjectId, kol[i]);
            unchecked {
                ++i;
            }
        }
    }
```
  