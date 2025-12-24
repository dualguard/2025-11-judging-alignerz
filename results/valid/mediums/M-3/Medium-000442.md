# [000442] `getUnclaimedAmounts` will always return the full amount of a user's NFT because `allocationOf` is never updated
  
  ### Summary

The protocol allows users to earn NFTs in order to later claim dividends. The function `getUnclaimedAmounts` prepares the distribution of dividends by calculating how much is the allocation of an NFT, and adjusts it by the seconds already claimed. 

However, since `allocationOf` is never updated, the claimed seconds will always be 0 and allow users to claim 100% of their original amount of dividends even though they claim all of it.

### Root Cause

`claimTokens` does not update `allocationOf`.

### Attack Path

1. Alice earns an NFT with amount = `100e6`
2. Alice calls `claimTokens` when 99% of the vesting period has passed, withdrawing 99% of her vesting tokens.
3. Because `allocationOf` is not updated, a dividend distribution will calculate her amount to be as if she didn't claim any seconds of it, earning more dividends than intended

### Impact

User steal dividends from other rightful users

### Mitigation

After updating the allocation storage of the bidding or reward project, assign the new value to `allocationOf`.

  