# [000450] Users splitting their TVS will loose their allocations
  
  ### Summary

1/ splitting TVS updates first the main NFT allocation (reduces it)
2/ All subsequent NFT created will have an amount assigned based on the first NFT's new value, loosing funds


The protocol allows NFT holders to split their NFTs, if they want for example to send a part of their rewards to someone else. The function `splitTVS` takes into parameter an array of `percentages` that represent the percentage of amount split to be set to each NFT, with the first element of the array being the percentage that should stay in the NFT we're splitting.

However, since the for loop first changes the storage of the amount of the first NFT, all NFTs that will be created will have their amount split from a lower value than intended, resulting in a loss of funds for users.

### Root Cause

`splitTVS` calculations are based on the main NFT `amount` **after** its new percentage is set rather than before.

### Attack Path

1. Alice is an investor who has an NFT with amount `1000e6`
2. She wants to share it equally with 10 people who helped investing
3. Alice calls `splitTVS` with `percentages` being an array of 10 values containing `1000` (10% of `BASIS_POINT`)
4. The first NFT has its new allocation set in storage to 10%, so it goes from `1000e6` to `100e6`
5. Every subsequent NFT created will have a value assigned to 10% of `100e6`, so they will receive `10e6`

Alice now owns a total of `190e6` accross all NFTs, and lost `810e6`.

### Impact

Loss of funds for users

### Mitigation

There are multiple ways of mitigating this issue.

You can update the storage of the main NFT to be the last in the loop, so other NFTs will not update from an incorrect amount.

You can also use a temporary variable that will hold the original value for the same desired effect.

  