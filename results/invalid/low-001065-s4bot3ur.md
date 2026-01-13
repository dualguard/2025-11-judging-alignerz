# [001065] Protocol Applies Merge Fees Despite Documentation Claiming “No Merging Fee
  
  ### Summary

The Alignerz documentation explicitly states that **merging TVSs does not incur any fee**, but the actual implementation applies a fee whenever a merge operation occurs. During initilization the fee is not set but it can be set by admin whenever admin wants
This creates a mismatch between expected and actual behavior, users incur token loss (if fee is set) during merges even though the documentation promises fee-free merging.

### Root Cause

https://drive.google.com/file/d/1xN5bYUPd_BkBMtoRruHEO1CBUx0vBiit/edit 

The documentation (pg-12) says merging TVSs is free.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1013

But the `mergeTVS()` function internally calls:
```solidity
(uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
```
This applies a fee (using splitFeeRate), reducing the user's amounts during merging.
Because splitFeeRate is not initialized , no fee is charged until the admin sets a fee. But the documentation claims merging is always free.


### Internal Pre-conditions

1. Admin must set a non-zero splitFeeRate.
2. User must execute `mergeTVS()`.
3. TVS must contain multiple flows (normal behavior).

### External Pre-conditions

_No response_

### Attack Path

1. Users read documentation claiming merge is free.
2. Admin later sets a non-zero merge fee.
3. User merges TVSs expecting no fee.
4. User unknowingly loses a portion of tokens due to fee deduction.
5. User may misinterpret this as protocol malfunction.

### Impact

If merge fee is set then user will consider it as loss because the documentation claims that merge fee rate is zero.

### PoC

_No response_

### Mitigation

Either change the documentation or remove the merge fee logic
  