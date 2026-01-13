# [000449] Reentrancy issue in `splitTVS` allows attacker to inflate their real TVS
  
  ### Summary

The function `splitTVS` allows users to split their TVS into sub parts. An invariant is that the sum of all amounts in the original NFT and the ones created must be less or equal than the original NFT amount.

But the `for` loop has an external call when creating NFTs. If the address we are minting the new NFT to is a contract, then the safe minting function will call `onERC721Received` on the receiving contract. A malicious contract can profit from this by updating the allocation of the original NFT during the reentrancy (before the values are set to the newly created NFT), which will result in sending a bigger allocation to the new NFT than what it's supposed to receive, breaking the invariant.

### Root Cause

Re-entrancy in [`splitTVS`](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1086) allows attacker to inflate the allocation set.

### Attack Path

1. Attacker has 2 NFT in a contract to prepare the attack. Both NFTs have an amount of `100e6`
2. Attacker calls `splitTVS` for his first NFT with the parameter `percentages` set to `[0, 50%, 50%]`, which means that the original NFT should have 0% of its original value by the end of the transaction, and 2 new NFTs should be created with a value of 50% of the original NFT's amount.
3. In the first loop, the contract sets the original NFT value to 0, which is intended.
4. In the second loop, a new NFT is minted to the attacker, which triggers the contracts' function `onERC721Received`.
5. During this call, the contract calls `mergeTVS`, where we merge the attackers `nft2` into the first NFT used in step 2. This results in having `nft1.amount = 100e6`.
6. The function `splitTVS` recovers execution, sets the value of `newNft1` to 50% of `nft1`, so `50e6`.
7. When the next NFT is created, the same re-entrancy occurs, with our attacker merging `newNft1` into `nft1`, which has now an amount of `150e6`.
8. The function `splitTVS` recovers execution, sets the value of `newNft2` to 50% of `nft1`, so `75e6`.

Our attack originally had `100e6 + 100e6 = 200e6` accross both his NFTs. He now has `150e6 + 75e6 = 225e6`, hence inflating his original amount.

### Impact

This inflation of value can be used to drain the contract of funds, for example during dividends distribution.

### Mitigation

Either implement CEI pattern to prevent re-entrancy from having an effect by sending all NFT created at the end of `splitTVS`, or use a reentrancy guard accross `splitTVS` and `mergeTVS`.

  