# [001009] Users may lose value with each token claiming due to rounding down in `getClaimableAmountAndSeconds()`
  
  ### Summary


Due to rounding down precision loss in [getClaimableAmountAndSeconds()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L993) users claim less tokens. 

### Root Cause

Protocol incentivize users to created bids with longer vesting period of time. The user chosen vesting period is saved in [bids](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L731) struct: 
```solidity
    function placeBid(uint256 projectId, uint256 amount, uint256 vestingPeriod) external {
//...
        biddingProject.bids[msg.sender] =
            Bid({amount: amount, vestingPeriod: vestingPeriod}); // @audit store the user's vesting period
        biddingProject.totalStablecoinBalance += amount;


        emit BidPlaced(projectId, msg.sender, amount, vestingPeriod);
    }
```
When user claim the NFT, the value of `vestingPeriod` he choose, is saved in his NFT flow. 
As a result NFT can have flows with very long vesting period
```solidity
    function claimNFT(uint256 projectId, uint256 poolId, uint256 amount, bytes32[] calldata merkleProof) external returns (uint256) {
//...
        Bid storage bid = biddingProject.bids[msg.sender];
       
        biddingProject.allocations[nftId].amounts.push(amount);
        biddingProject.allocations[nftId].vestingPeriods.push(bid.vestingPeriod); // @audit save vesting period to NFT
//...
}
```
The problem is when user [claims vested](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L993) tokens, the `amount * claimableSeconds` is divided by `vestingPeriod`. 
Since `vestingPeriod` can be very large the loss due to rounding down  can be semnificative. 

```solidity
    function getClaimableAmountAndSeconds(Allocation memory allocation, uint256 flowIndex) public view returns(uint256 claimableAmount, uint256 claimableSeconds) {
//...
        uint256 vestingPeriod = allocation.vestingPeriods[flowIndex];
//...
        claimableAmount = (amount * claimableSeconds) / vestingPeriod; // @audit loss due to rounding down
//...
```

### Internal Pre-conditions

None

### External Pre-conditions

1. User set a very long `vestingPeriod`. 

### Attack Path

1. Alice creates a bid and mint a NFT with very large `vestingPeriod`
2. Alice calls `claimTokens():  
- the `amount * claimableSeconds` is equal to  `x * vestingPeriod - 1`. 
- Alice receive ` (x-1) * vestingPeriod` tokens
- Alice looses ` vestingPeriod - 1` tokens due to rounding

### Impact

Users may loose tokens each time they claim tokens. The amount lost may be significant for very large vesting periods.

### PoC

_No response_

### Mitigation

Consider enforcing a max value for `vestingPeriod` (eg. 10 years) and a minimum USD amount when bids are placed (eg. 10 USD). 
Doing this you limit the value users may lose due to rounding down in `claimTokens`. 

  