# [001005] Compounding fee miscalculations lead to users losing more tokens than the stated fee rate
  
  ### Summary

The incorrect use of cumulative feeAmount instead of individual flow fees in [FeesManager.sol::calculateFeeAndNewAmountForOneTVS() ](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L172) will cause exponential token loss for users as each subsequent vesting flow subtracts the accumulated fees from all previous flows, not just its own fee.

### Root Cause

In `FeesManager.sol:182` the function subtracts the cumulative feeAmount (which accumulates across all flows) instead of calculating and subtracting only the individual fee for each flow:

```solidity 
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) 
    public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);  // Accumulates total fee
        // @audit Bug: Subtracts cumulative fee instead of individual flow fee
        newAmounts[i] = amounts[i] - feeAmount;  // Each flow loses MORE than it should
    }
}
```


### Internal Pre-conditions

1. User needs to have a TVS NFT with multiple vesting flows (at least 2 flows)
2. User calls mergeTVS() or splitTVS() triggering fee calculation
3. mergeFeeRate or splitFeeRate needs to be non-zero

### External Pre-conditions

None

### Attack Path

- User owns TVS NFT with 3 flows: [1000, 2000, 3000 tokens] = 6000 total
- User calls splitTVS() with 1% fee rate (100 basis points)
- Function processes Flow 0: fee = 10, new amount = 1000 - 10 = 990 ✓
- Function processes Flow 1: cumulative fee = 30, new amount = 2000 - 30 = 1970 ❌ (should be 1980)
- Function processes Flow 2: cumulative fee = 60, new amount = 3000 - 60 = 2940 ❌ (should be 2970)
- User receives 5900 tokens instead of 5940 tokens
- User loses 40 extra tokens 

### Impact

Progressive token theft from users
Protocol/treasury receives more fees than configured, breaking fee promises to users

### PoC

_No response_

### Mitigation

```solidity 
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) 
    public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    newAmounts = new uint256[](length);
    
    for (uint256 i; i < length;) {
        // Calculate individual fee for THIS flow only
        uint256 individualFee = calculateFeeAmount(feeRate, amounts[i]);
        
        // Accumulate total fee (for tracking/treasury transfer)
        feeAmount += individualFee;
        
        // Fix: Subtract only THIS flow's fee, not cumulative
        newAmounts[i] = amounts[i] - individualFee;
        
        unchecked { ++i; }
    }
}
```
  