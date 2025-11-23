# [000092] Users pay `updateBidFee` for no-op bid updates
  
  ### Summary

The lack of change validation in `updateBid()` will cause users to waste gas and pay unnecessary fees as they will call `updateBid()` with the same values as their existing bid, resulting in no actual state change but still charging the `updateBidFee`.

### Root Cause

In `AlignerzVesting.sol:741-776`, the `updateBid()` uses `>=` comparisons for both `newAmount` and `newVestingPeriod `validation, allowing values equal to the current bid parameters.  The function does not check whether any actual change is being made before charging the `updateBidFee`, allowing users to pay fees for no-op updates.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L752-L754

### Internal Pre-conditions

1. User needs to have an existing bid via `placeBid()`
2. Owner needs to have set `updateBidFee > 0` via `setUpdateBidFee()`

### External Pre-conditions

N/A

### Attack Path

1. User places initial bid of 1000 USDT with 90 days vesting period
2. User calls `updateBid(projectId, 1000, 90 days)` with the exact same parameters
3. Function validates `1000 >= 1000` and `90 days >= 90` days - both pass 
4. No additional stablecoin is transferred (since amounts are equal)
5. User still pays `updateBidFee` to treasury 
6. Bid values are set to the same values they already had

### Impact

Users waste gas and pay the `updateBidFee` for transaction that make no actual changes to their bids. While is this a primarly user error, the protocol could prevent this loss to the user by require at least one parameter to increase. This is could lead to poor UX that allows users to accidentally waste funds on meaningless transactions.

### PoC

Not applicable - the issue is straightforward from code inspection.

### Mitigation

Just add a validation to ensure that at least one parameter actaully changes:

```diff
function updateBid(uint256 projectId, uint256 newAmount, uint256 newVestingPeriod) external {  
    require(projectId < biddingProjectCount, Invalid_Project_Id());  
    BiddingProject storage biddingProject = biddingProjects[projectId];  
    require(  
        block.timestamp >= biddingProject.startTime &&  
            block.timestamp <= biddingProject.endTime &&  
            !biddingProject.closed,  
        Bidding_Period_Is_Not_Active()  
    );  
  
    Bid storage bid = biddingProject.bids[msg.sender];  
    uint256 oldAmount = bid.amount;  
    require(oldAmount > 0, No_Bid_Found());  
    require(newAmount >= oldAmount, New_Bid_Cannot_Be_Smaller());  
    require(newVestingPeriod > 0, Zero_Value());  
    require(newVestingPeriod >= bid.vestingPeriod, New_Vesting_Period_Cannot_Be_Smaller());  
      
+   // Add check to prevent no-op updates  
+   require(newAmount > oldAmount || newVestingPeriod > bid.vestingPeriod, "No_Changes_To_Update");  
  
    // ... rest of function  
}
```
  