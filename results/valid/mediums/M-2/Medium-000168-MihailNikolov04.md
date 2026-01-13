# [000168] Fees are hardcoded to the max of 10 cents instead of 1 dollar
  
  ### Summary

As mentioned in the readme:
>Q: Are there any limitations on values set by admins (or other roles) in protocols you integrate with, including restrictions on array lengths?
bidFee and updateBidFee can have a maximum of 1 USDC/USDT

This contradicts the hardcoded values in the `FeesManager` contracts as can be seen here:
```solidity
   function _setUpdateBidFee(uint256 newUpdateBidFee) internal {
@>        require(newUpdateBidFee < 100001, "Bid update fee too high"); <@

        uint256 oldUpdateBidFee = updateBidFee;
        updateBidFee = newUpdateBidFee;

        emit updateBidFeeUpdated(oldUpdateBidFee, newUpdateBidFee);
    }

   function _setBidFee(uint256 newBidFee) internal {
        // Example placeholder limits
@>        require(newBidFee < 100001, "Bid fee too high"); <@

        uint256 oldBidFee = bidFee;
        bidFee = newBidFee;

        emit bidFeeUpdated(oldBidFee, newBidFee);
    }
```
The hardcoded values represent 10 cents in both USDC and USDT (As 1 dollar represents `1 * 10^6` in those currencies). This directly contradicts the `ReadMe` and should be accepted as at least valid medium due to the following reasons:
1. The owner is capped to set the limit to 10 cents which means at least 90% loss on fees (accounting that fee is intended to be set to 1 usd)
2. Likelihood on this one is high because it practically occurs every time a user place a bid or update one
3. Imagining a scenario with 1k bids, this results for a received fee of 100 USD instead of 1000 USD, which is a pretty significant loss 

### Root Cause

Wrongly hardcoded values in the [`FeesManager`](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L6) contract

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

Owner isn't able to set a fee higher than 10 cents

### Impact

Significant fee loss for the protocol 

### PoC

-

### Mitigation

Change the hardcoded values so that they represent 1 USD
  