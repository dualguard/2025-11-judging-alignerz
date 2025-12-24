# [000457] endTimeHash is not used to enforce bid time correctly
  
  ### Summary

As per the whitepapers:

> To prevent last-minute bid manipulation, bidding window will be hashed. After the bidding ends, we will reveal the exact input used to generate the hash, ensuring fairness.

What is supposed to happen under the hood:

1. A new project launch. The `biddingProject` has an `endTime`, and an `endTimeHash`
2. Users are allowed to bid up until `endTime`. `placeBid` and `updateBid`  will update `biddingProject.bids[msg.sender]`
3. After the bidding period ends, the owner reveals the exact time the bidding really ended (by showing the value used to create `endTimeHash`). We will call this value `realEndtime`, and it must be `<= endTime`
4. The protocol admin generate all winners from bids that occured BEFORE `realEndTime`, even though users were allowed to bid after (this is to avoid malicious use of the bidding system)
5. After the merkle tree are set, users can only claim amounts from the values chosen by the admin

However, when they call `claimNFT`, users' allocations will have their vesting periods being equal to the value of their last bid, even though it happens after `realEndTime`. [(affected line)](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L880)

This will cause users to have longer vesting times than what their last valid bid suggests.

### Root Cause

The last bid's vesting period is saved to an NFT, while it should be the last valid bid (before `endTimeHash`)

### Internal Pre-conditions

A winning user must update their bid to a higher vesting period after `realEndTime`

### Impact

User receive their vesting rewards after a longer time.

### Mitigation

The best refactoring tip that I would give would be to get rid of `endTime` and have the admin call a new function `stopBids` by giving a variable `uint256` showing the timestamp of the expected end time, and a salt to not make the value obvious.

The function checks that the block.timestamp is bigger/equal than the end time, and that the hash matches. It then close the project, so users can't make bigger bids. The end time is still a surprise because users don't know when the project will stop.

Another possible fix would be to add the vesting period to the hash when users claim their NFT. This way, it is the admins that set the value for it, and not the latest _potentially invalid_ bid.

  