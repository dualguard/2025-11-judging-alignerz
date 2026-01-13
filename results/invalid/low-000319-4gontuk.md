# [000319] Stale bid records after NFT/refund claim in AlignerzVesting
  
  ### Description:
In the bidding path of `AlignerzVesting`, user bids are stored in `biddingProject.bids[msg.sender]` and later consumed during claiming. For example, in `claimNFT()`:

```solidity
Bid storage bid = biddingProject.bids[msg.sender];
require(bid.amount > 0, No_Bid_Found());
// ...
uint256 nftId = nftContract.mint(msg.sender);
biddingProject.allocations[nftId].amounts.push(amount);
biddingProject.allocations[nftId].vestingPeriods.push(bid.vestingPeriod);
// ...
```

and in [`claimRefund()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L835-L853):

```solidity
Bid storage bid = biddingProject.bids[msg.sender];
require(bid.amount > 0, No_Bid_Found());
// ...
biddingProject.totalStablecoinBalance -= amount;
biddingProject.stablecoin.safeTransfer(msg.sender, amount);
```

After a successful NFT claim or refund, the corresponding `Bid` struct is **not deleted** from `bids[msg.sender]`. Although this does not break current logic (since `placeBid()` is only usable while the project is open and gated by `biddingProject.closed`), it leaves stale bid data in storage:

- Off‑chain indexers may misinterpret `bid.amount` as an “active bid” even after the user has fully claimed.
- Future extensions relying on `bids[msg.sender]` being zeroed after claim could misbehave.

### Impact:
Stale `bids[msg.sender]` structs remain in storage after successful claim, slightly increasing storage usage and risking confusion for off‑chain tools or future logic.

### Recommendation:
Clear the user’s bid record once it has been fully consumed, for example:

```solidity
function claimNFT(...) external returns (uint256) {
    BiddingProject storage biddingProject = biddingProjects[projectId];
    Bid storage bid = biddingProject.bids[msg.sender];
    require(bid.amount > 0, No_Bid_Found());
    // existing claim logic...
    delete biddingProject.bids[msg.sender];
    return nftId;
}

function claimRefund(...) external {
    BiddingProject storage biddingProject = biddingProjects[projectId];
    Bid storage bid = biddingProject.bids[msg.sender];
    require(bid.amount > 0, No_Bid_Found());
    // existing refund logic...
    delete biddingProject.bids[msg.sender];
}
```

This ensures the `bids` mapping reflects only outstanding bids and avoids stale state after claims.

  