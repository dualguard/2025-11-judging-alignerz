# [000061] Public Visibility of endTime in biddingProjects Mapping Defeats Anti-Sniping Mechanism
  
  ## Summary

The [`biddingProjects`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L45) mapping is declared as `public`, exposing the `endTime` field to all users. This directly contradicts the whitepaper's anti-sniping design where the bidding window is hashed to hide the exact end time. Any user can simply call `biddingProjects[projectId].endTime` to read the precise auction end timestamp, rendering the `endTimeHash` privacy mechanism completely useless.

## Vulnerability Details

```solidity
function launchBiddingProject(..., uint256 endTime, bytes32 endTimeHash, ...) {
    biddingProject.endTime = endTime;  // ← Readable via public getter
}
```

The whitepaper describes showing users only a hash like `"7d1ae2ed5171c06d670f4aa30b595712bc81ba0e22a8811fa9eedc5bb3a236fd"` to prevent them from knowing when to snipe. But since `endTime` is not hased anyone can see it.

## Impact

### Broken Anti-Sniping Mechanism

The whitepaper's anti-sniping mechanism is completely ineffective:

- Sniping bots can read the exact end time from public storage
- Users can target the precise moment to submit last-second bids


## Recommended Mitigation

Impliment the endTimeHash mechanism as described in the whitepaper

  