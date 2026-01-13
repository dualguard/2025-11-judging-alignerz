# [000055] Unable to set bid fee more than 0.1 usdc/usdt due to wrong checking condition
  
  ### Summary

Wrong require condition when setting and update bid fee, making setting fee greater than 0.1 usdt/usdc is impossible

### Root Cause

From Q&A:

`Q: Are there any limitations on values set by admins (or other roles) in protocols you integrate with, including restrictions on array lengths?`
`bidFee and updateBidFee can have a maximum of 1 USDC/USDT`

But when set and update fee, maximum fee can be set is only 100000, which equal to 0.1 usdc/usdt due to these tokens have 6 decimals:

    function _setBidFee(uint256 newBidFee) internal {
        // Example placeholder limits
        require(newBidFee < 100001, "Bid fee too high");     <@

        uint256 oldBidFee = bidFee;
        bidFee = newBidFee;

        emit bidFeeUpdated(oldBidFee, newBidFee);
    }

    function _setUpdateBidFee(uint256 newUpdateBidFee) internal {
        require(newUpdateBidFee < 100001, "Bid update fee too high");    <@

        uint256 oldUpdateBidFee = updateBidFee;
        updateBidFee = newUpdateBidFee;

        emit updateBidFeeUpdated(oldUpdateBidFee, newUpdateBidFee);
    }

Which making setting bid fee greater than 0.1 usdt/usdc impossible

### Internal Pre-conditions

none

### External Pre-conditions

none

### Attack Path

Owner try to set bid fee greater than 100000 (0.1 usdt/usdc) and fail

### Impact

Loss of fee due to unable to set fee higher than 100000

### PoC

_No response_

### Mitigation

_No response_
  