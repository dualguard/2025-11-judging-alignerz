# [000593] Incorrect Comparison Operator in Claim Functions Reduces User Claim Window by Preventing Claims at Exact Deadline Timestamp
  
  ## Summary

Using `<` instead of `<=` in `claimRewardTVS` and `claimStablecoinAllocation` prevents users from claiming at the exact `claimDeadline`, reducing the claim window. This contradicts the variable name and the owner distribution logic, which uses `>` to allow distribution only after the deadline.

## Root Cause

In [`AlignerzVesting.sol:508`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L508) and `AlignerzVesting.sol:517`, the comparison uses `block.timestamp < rewardProject.claimDeadline` instead of `block.timestamp <= rewardProject.claimDeadline`. This excludes the exact deadline timestamp from the claim window. The variable name `claimDeadline` and the owner distribution functions (which use `block.timestamp > rewardProject.claimDeadline`) indicate users should be able to claim up to and including the deadline.
```js
    function distributeStablecoinAllocation(...) external {
        // ...
        require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
        // ...
    }
```

## Internal Pre-conditions

1. A reward project must exist with `rewardProjectId` set
2. The project must have a `claimDeadline` set via `launchRewardProject`
3. KOLs must have allocations set via `setTVSAllocation` or `setStablecoinAllocation`
4. `block.timestamp` must equal `rewardProject.claimDeadline` (the edge case where the bug manifests)

## External Pre-conditions

None

## Attack Path

This is a vulnerability path, not an attack path:

1. A reward project is launched with a `claimDeadline` set to a specific timestamp
2. KOLs attempt to claim their TVS or stablecoin allocations
3. When `block.timestamp == claimDeadline`, the `require(block.timestamp < rewardProject.claimDeadline, Deadline_Has_Passed())` check fails
4. Users cannot claim at the exact deadline timestamp, losing one second (or one block) of the intended claim window
5. After the deadline passes (`block.timestamp > claimDeadline`), only the owner can distribute unclaimed tokens

## Impact

Users cannot claim their TVS or stablecoin allocations at the exact deadline timestamp, reducing the claim window by one block. This can cause legitimate claims to be rejected if they occur exactly at the deadline, forcing users to rely on the owner's distribution function instead of claiming directly.

## Proof of Concept
N/A

## Mitigation

Change the comparison operator from `<` to `<=` in both functions.
  