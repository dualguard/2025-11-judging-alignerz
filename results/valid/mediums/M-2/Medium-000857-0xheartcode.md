# [000857] Fee Limits Implementation Does Not Match Documentation
  
  ## Summary

The implemented fee validation limits in `FeesManager.sol` are 10x more restrictive than documented specifications, creating a discrepancy between actual contract behavior and expected limits.

## Vulnerability Detail

The fee validation implementation differs significantly from documentation:

**Location**: [`FeesManager.sol:86, 111`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L86)

```solidity
function _setBidFee(uint256 newBidFee) internal {
    require(newBidFee < 100001, "Bid fee too high");  // Actual limit: 100,000
    // ...
}

function _setUpdateBidFee(uint256 newUpdateBidFee) internal {
    require(newUpdateBidFee < 100001, "Bid update fee too high");  // Actual limit: 100,000
    // ...
}
```

**Documentation Claims**: "bidFee and updateBidFee can have a maximum of 1 USDC/USDT"
- **Expected Limit**: 1,000,000 units (6-decimal tokens)
- **Actual Limit**: 100,000 units (10x lower)

**Root Cause**: Implementation uses different limit values than specified in documentation.

## Impact

**Low Severity**: Documentation mismatch affecting integrations and expectations

- **Integration Confusion**: Developers may assume higher fee limits are possible

## Code Snippet

**Current implementation**:
```solidity
// FeesManager.sol:86, 111
require(newBidFee < 100001, "Bid fee too high");
require(newUpdateBidFee < 100001, "Bid update fee too high");
```

**Expected based on documentation**:
```solidity
require(newBidFee < 1000001, "Bid fee too high");     // 1 USDT/USDC
require(newUpdateBidFee < 1000001, "Bid update fee too high");
```


  