# [000354] BidFee and UpdateFee can not be set to the maximum value has stated in the README
  
  ### Summary

According to the README, a maximum value of 1 dollar (1e6) in stable coin value can be set for bid fee and update fee but this is false as the max value allowed by the code is 0.1usd (1e5). 

> Q: Are there any limitations on values set by admins (or other roles) in protocols you integrate with, including restrictions on array lengths?
bidFee and updateBidFee can have a maximum of 1 USDC/USDT





### Root Cause

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L86-L89

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L111-L114



```solidity
    function _setBidFee(uint256 newBidFee) internal {
        // Example placeholder limits

@audit>>        require(newBidFee < 100001, "Bid fee too high"); // max of 1usdt not 0.1usdt

        uint256 oldBidFee = bidFee;
        bidFee = newBidFee;

        emit bidFeeUpdated(oldBidFee, newBidFee);
    }

```

```solidity
    function _setUpdateBidFee(uint256 newUpdateBidFee) internal {

@audit>>        require(newUpdateBidFee < 100001, "Bid update fee too high"); // i think this should be 1000001 bug low maybe no impact cannot set fee to 1 usdt 1e6 according to the readme....... note low 

        uint256 oldUpdateBidFee = updateBidFee;
        updateBidFee = newUpdateBidFee;

        emit updateBidFeeUpdated(oldUpdateBidFee, newUpdateBidFee);
    }
```

This shows a low severity bug that prevents admin from setting the max value has stated in the readme.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

Require check present prevent the admin from setting the max valuation of bidfee and updatedfee

### Impact

Inability to charge the max fee

### PoC

_No response_

### Mitigation

Change the required checked to accommodate values from 0 - 1e6.
  