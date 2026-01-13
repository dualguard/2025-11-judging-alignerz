# [000592] Missing `onlyOwner` Access Control on `distributeStablecoinAllocation` Allows Unauthorized Distribution of Unclaimed Stablecoin Tokens
  
  
## Summary

The missing `onlyOwner` modifier on `distributeStablecoinAllocation` allows anyone to distribute unclaimed stablecoin tokens for any reward project after the claim deadline, bypassing owner control over timing and distribution.

## Root Cause

In [`AlignerzVesting.sol:540`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L540), `distributeStablecoinAllocation` lacks the `onlyOwner` modifier. The comment states "Allows the owner to distribute...", but the function only checks that the claim deadline has passed, allowing any caller to trigger distribution.

```js
    function distributeStablecoinAllocation(...) external {
        // ...
        require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
        // ...
    }
```

## Internal Pre-conditions

1. A reward project must exist with `rewardProjectId` set
2. Stablecoin allocations must be set for KOLs via `setStablecoinAllocation`
3. The claim deadline (`rewardProject.claimDeadline`) must have passed
4. There must be unclaimed stablecoin allocations in `rewardProject.kolStablecoinAddresses`

## External Pre-conditions

None

## Attack Path

1. User waits for `block.timestamp > rewardProject.claimDeadline`
2. User calls `distributeStablecoinAllocation(rewardProjectId, kol)` with an array of KOL addresses
3. The function iterates and calls `_claimStablecoinAllocation` for each KOL, transferring stablecoin tokens
4. Distribution occurs without owner authorization

## Impact

The protocol loses control over when unclaimed stablecoin tokens are distributed. An attacker can force distribution at any time after the deadline, disrupting planned timing and potentially causing operational or financial issues. KOLs receive tokens earlier than intended, and the owner cannot delay or batch distributions as planned.

## Proof of Concept
N/A

## Mitigation

Add the `onlyOwner` modifier to `distributeStablecoinAllocation`:

```diff
+ function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external onlyOwner {
}
```
  