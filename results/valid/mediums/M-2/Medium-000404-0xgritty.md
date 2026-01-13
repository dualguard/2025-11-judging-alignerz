# [000404] bidFee and updateBidFee can't be set upto 1 USDC/USDT
  
  ### Summary

- Readme states owner can set bidFee and updateBidFee upto 1 USDC, which is not true due to faulty require statement in FeesManager::_setBidFee.

- Which forces the max fees allowed to only .1 USDC which is like 10% of the amount stated in readme.


### Root Cause

- [Require](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L86) statement inside FeesManager::_setBidFee checks newBidFee against 100001 instead of 1000001. 


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

- Owner tries to set bidFee to 1e6 USDC and tx reverts due to newBidFee being greater than 100001.


### Impact

- Protocol won't be able to set bidFee and updateBidFee upto max 1 USDC/USDT and will lose on fees.


### PoC

_No response_

### Mitigation

_No response_
  