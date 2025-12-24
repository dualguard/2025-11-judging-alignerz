# [000541] Incorrect updateBidFee limit enforces 0.1 USDC instead of documented 1 USDC
  
  ### Summary

The mismatch between the README specification (maximum updateBidFee = **1 USDC = 1e6 units**) and the contract's validation (`newUpdateBidFee < 100001`) will cause **protocol misconfiguration** as admins will be unable to set a bid update fee up to the documented maximum. The contract currently restricts the fee to ~0.1 USDC, contradicting the intended design.


### Root Cause

In the [`_setUpdateBidFee`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L100-L117) function:

```solidity
function _setUpdateBidFee(uint256 newUpdateBidFee) internal {
    require(newUpdateBidFee < 100001, "Bid update fee too high");
    
    uint256 oldUpdateBidFee = updateBidFee;
    updateBidFee = newUpdateBidFee;
    
    emit updateBidFeeUpdated(oldUpdateBidFee, newUpdateBidFee);
}
```

This enforces a maximum of:

```
100,001 units ≈ 0.100001 USDC (USDC has 6 decimals)
```

But the README clearly states:

> "bidFee and updateBidFee can have a maximum of **1 USDC/USDT**"

1 USDC = **1,000,000 units (1e6)**, not 100,001.

**Root cause:** The contract enforces the wrong maximum bid update fee, limiting it to ~0.1 USDC instead of the intended 1 USDC.

### Internal Pre-conditions

1. Admin attempts to set updateBidFee to any value between **100,001** and **1,000,000** (inclusive), which is a valid and documented range.
2. `_setUpdateBidFee()` reverts because the contract incorrectly caps the fee at ~0.1 USDC.

### External Pre-conditions

_No response_

### Attack Path


1. Admin attempts to set update fee to a valid value like `400,000` (0.4 USDC).
2. Contract rejects it due to `require(newUpdateBidFee < 100001)`.
3. Admin cannot configure the protocol according to specifications.

### Impact

Protocol is stuck at setting fee at 0.1 usdc as max instead of documented 1USDC causing them to lose some funds. Since they should obviously be able to set it higher

### PoC

_No response_

### Mitigation

define a constant for consistency:

```solidity
uint256 constant MAX_BID_FEE = 1e6; // 1 USDC for 6-decimal tokens

function _setUpdateBidFee(uint256 newUpdateBidFee) internal {
    require(newUpdateBidFee <= MAX_BID_FEE, "Bid update fee too high");
    
    uint256 oldUpdateBidFee = updateBidFee;
    updateBidFee = newUpdateBidFee;
    
    emit updateBidFeeUpdated(oldUpdateBidFee, newUpdateBidFee);
}
```
  