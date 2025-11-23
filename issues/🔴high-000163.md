# [000163] User is charged fee for claimed flows when merging and splitting
  
  ### Summary

As can be seen in the [`Vesting::mergeTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002) function, the whole `amounts` is extracted form the allocation and provided as input on the `calculateFeeAndNewAmountForOneTVS` function, which means that this fee is applied on even on the fully collected flows. This an issue because when fees are applied on the fully collected flows, the user doesn't actually suffer a loss himself (since he already claimed everything that a fee can be applied for). Instead the other users of the protocol and the protocol itself suffer from this since the funds are transferred out of the protocol and there is nobody to cover it with their allocation amount being reduced. Malicious user can also make an exploit from that where he leaves one very small remaining flow (so that his loss is minimal) and start spam calling the merge and split functions to forcefully transfer funds out of the protocol

Also as a secondery issue comming from this is the following scenario:
1. User has an NFT with value of 1000 and he claims half of it (so 500 remains)
2. He splits the NFT on 2 parts and the max of 2% fee is applied (so now all of the NFT flows are worth 980 and it's claimable part is 490 split between the updated initial NFT and the newly minted one)
3. The problem here is that the user has already claimed half of the NFT meaning that he there will be 10 units loss for the protocol (as 500 + 490 = 990 and the fee that we applied is 20 which sums up for the total of 1010)

Overall the closer user is to the vesting period end, the lesser  he  suffers from the fees applied to his flows on split/merge

### Root Cause

Root cause of the issue is the fact that both `mergeTVS` and `splitTVS` functions apply fees to the already claimed  flows

### Internal Pre-conditions

User calls the `mergeTVS` or the `splitTVS` function

### External Pre-conditions

None

### Attack Path

The following attack path is possible:
1. User has an NFT with many fully claimed flows and small unclaimed ones
2. He starts spam calling the mentioned functions to transfer funds out of the protocol as he doesn't account the applied fees
3. Other users and the protocol itself suffer from the accounted fees, as there may not be enough funds to for claiming or performing other operations 

### Impact

The users and the protocol suffer a big losses and there may not be enough funds in the system for users to claim their winnings

### PoC

- 

### Mitigation

Don't enforce fees on the already claimed amounts as they are carried by the protocol and not by the actual user. enforce fees only on unclaimed amounts as they are not yet claimed, hence carried by the user
  