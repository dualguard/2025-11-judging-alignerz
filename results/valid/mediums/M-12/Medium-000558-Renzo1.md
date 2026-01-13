# [000558] Integer Division Rounding in `getClaimableAmountAndSeconds` Causes Permanent Claim Blocking for NFTs with Small Remaining Amounts
  
  
## Summary

Integer division rounding in `getClaimableAmountAndSeconds` will cause permanent claim blocking for NFT holders as the calculation rounds down to zero when the remaining claimable amount is smaller than the vesting period denominator, making affected NFTs permanently unclaimable.

## Root Cause

In [`AlignerzVesting.sol:994`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L993-L994), the `claimableAmount` calculation uses integer division that rounds down to zero:

```js
claimableAmount = (amount * claimableSeconds) / vestingPeriod;
require(claimableAmount > 0, No_Claimable_Tokens());
```

When `(amount * claimableSeconds) < vestingPeriod`, the division yields zero, causing the function to revert. This blocks `claimTokens` (line 959) since it calls `getClaimableAmountAndSeconds` and requires a non-zero claimable amount.

## Internal Pre-conditions

1. An NFT allocation must have a `vestingPeriod` larger than the product of the remaining `amount` and `claimableSeconds`.
2. The user must have claimed tokens (multiple times), reducing the remaining `claimableSeconds` to a small value. This can happen naturally (unintentionally) and intentionally.
3. The `amount` must be small enough that `amount * claimableSeconds < vestingPeriod`. This can occur when:
   - The original allocation had a small `amount` (low decimal tokens or small allocations)
   - The `amount` was reduced through `splitTVS` (line 1133), which divides amounts by `BASIS_POINT` (10,000), potentially creating very small amounts
   - The user is near the end of their vesting period with minimal remaining time

## External Pre-conditions

None required. This is a deterministic issue based on the contract's internal state.

## Attack Path
This can happen to an unknowing user such that they unintentional brick their NFT. However we want to use this sector to show how this can be used in a griefing context targeting an unknowing victim in the wild.

1. User places a bid with a large `vestingPeriod` (e.g., 1 year = 31,536,000 seconds) during bidding, or owner sets a large `vestingPeriod` for KOLs during reward project launch.
2. User receives an NFT with an allocation containing the large `vestingPeriod` and an `amount` that, when multiplied by small remaining `claimableSeconds`, is less than `vestingPeriod`.
3. User claims tokens multiple times, reducing `claimableSeconds` to a small value (e.g., 1-10 seconds remaining).
4. When `amount * claimableSeconds < vestingPeriod`, the next claim attempt calls `getClaimableAmountAndSeconds`, which calculates `claimableAmount = (amount * claimableSeconds) / vestingPeriod = 0`.
5. The function reverts with `No_Claimable_Tokens()` error, permanently blocking all future claims from this NFT.
6. (Optional griefing) User/Attacker sells the NFT to an unsuspecting buyer, who cannot claim the remaining tokens.

Alternative path (via splitting):
1. User splits their TVS using `splitTVS`, creating new NFTs with smaller `amounts` calculated as `(baseAmounts[j] * percentage) / BASIS_POINT` (line 1133).
2. If the split results in a very small `amount` (e.g., 1 wei or a few tokens for low decimal tokens), and the user has a large `vestingPeriod` with small remaining `claimableSeconds`, the same rounding issue occurs.
3. The affected NFT becomes permanently unclaimable.

## Impact

Affected NFT holders cannot claim any remaining tokens from their NFTs. The loss equals the unclaimed tokens locked in the contract. For griefing, the buyer/receiver suffers the loss of unclaimable tokens.

## Proof of Concept
N/A


## Mitigation

1. Remove the strict `require(claimableAmount > 0)` check and allow the function to return zero when appropriate, or
2. Add a minimum threshold check: if `claimableSeconds > 0` but `claimableAmount == 0`, calculate the remaining unclaimed amount differently (e.g., `remainingAmount = amount - (amount * claimedSeconds / vestingPeriod)`) and return that if it's greater than zero, or
3. Implement a dust collection mechanism that allows claiming the final remainder when `claimableSeconds > 0` but the calculated `claimableAmount` rounds to zero, ensuring all tokens can eventually be claimed.

  