# [000860] Owner Cannot Set Maximum Fee Amount as Specified in Documentation
  
  ### Summary

According to the protocol documentation, both `bidFee` and `updateBidFee` “can have a maximum of 1 USDC/USDT” (i.e., 1e6 when using 6-decimals). However, in the current implementation, the maximum value permitted by the require checks is 100001 (1e5 + 1), which is an order of magnitude lower than the documented limit.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L111

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L86

### Root Cause

The maximum fee constraint is incorrectly implemented at 1e5 instead of 1e6, preventing alignment with the documented design and reducing potential fee revenue.

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

None

### Impact

- The protocol collects less fee revenue than intended.
- Owner cannot configure fees according to documented specifications.

### PoC

None

### Mitigation

Update the require statements to allow a maximum of 1e6 + 1 instead of 1e5 + 1.
  