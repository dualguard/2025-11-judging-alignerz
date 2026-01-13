# [000296] Fee payment without bid success Guaranteed
  
   

**Summary:** Users are required to pay bid and update fees even when there's no guarantee their bids will be successful or result in token allocation, creating unfair economic conditions.

**Root Cause** In `AlignerzVesting.sol` contract,  the `placeBid` and `updateBid` function  is charging fees at the wrong point in the transaction lifecycle without conditional success checks.

**Vulnerability Details:** The contract charges fees at the time of bid placement and update, before any allocation is guaranteed:
```javascript
    // In placeBid: fee charged immediately
    if (bidFee > 0) {
        biddingProject.stablecoin.safeTransferFrom(msg.sender, treasury, bidFee);
    }

    // In updateBid: fee charged immediately  
    if (updateBidFee > 0) {
        biddingProject.stablecoin.safeTransferFrom(msg.sender, treasury, updateBidFee);
    }
```
These fees are paid regardless of whether the user receives any token allocation,the project is successfully finalized,the user's bid meets allocation criteria which doesn't favour the users as this lead to unfair economics to the users and also make them have trust issues with the protocol.

And even though we have the `claimRefund` function, it doesn't include the `bidFee` or `updateBidFee` refund that has already been collected earlier, and this doesn't favour the users.

**Impact:**
1. Users pay fees for potentially zero benefit
2. Creates economic barrier to participation
3. May lead to user funds being wasted
4. Could be considered predatory fee structure
5. Reduces trust in the platform

**Tool Used:** Manual review

**Affected Line of Code:** https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L727

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L767

**POC:**

**Recommended Mitigation** Inside the `claimNFT` function let there be a mechanism that allows users to deduct fee from users when they are about to claim, or inside the `finilizeBid` function there should be a mechanism that allows users to claim back their fee if they don't have an allocation.
  