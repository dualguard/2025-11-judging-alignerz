# [000378] Missing Post-Deadline NFT Recovery Mechanism Causing Permanent Token Lock in Bidding Projects
  
  ## Summary

The AlignerzVesting contract lacks an owner function to collect unclaimed NFTs from bidding projects after the claim deadline expires. While reward projects have recovery mechanisms (`distributeRemainingRewardTVS`, `distributeRemainingStablecoinAllocation`), bidding projects have no equivalent function. Tokens from Users who win allocations but fail to claim NFTs within the deadline will permanently be stuck.

## Root Cause

Once `claimDeadline` passes, users cannot call `claimNFT()` and owner has no function to mint NFTs on behalf of users. Token allocations remain in contract storage but are inaccessible.

## Impact

Unclaimed TVS with their token amount will be permanently stuck.

## Attack Path

1. User places winning bid in bidding project
2. Bidding closes, user is allocated tokens in merkle tree
3. User didn't claimed within the deadline.
4. `claimDeadline` passes
5. User attempts `claimNFT()` → Reverts with `Deadline_Has_Passed()`
6. `No recovery mechanism exists to get those, not even for the owner.`

## Mitigation

**Recommended Solution: Add Owner Function to claim Unclaimed NFTs**
or 
Implement an owner function similar to `distributeRemainingRewardTVS`:

  