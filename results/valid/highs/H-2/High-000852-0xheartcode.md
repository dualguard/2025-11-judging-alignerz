# [000852] Array Initialization Missing in Fee Calculation Breaks Split/Merge Operations
  
    ## Summary

  The `calculateFeeAndNewAmountForOneTVS` function in `FeesManager.sol` attempts to assign values
  to a zero-length return variable array, causing complete failure of TVS split and merge
  operations due to out-of-bounds access.

  ## Vulnerability Detail

  The fee calculation function declares a return variable array but fails to initialize it with
  proper length before attempting to assign values:

  **Location**: [`FeesManager.sol:168-173`](https://github.com/dualguard/2025-11-alignerz/tree/main
  /protocol/src/contracts/vesting/feesManager/FeesManager.sol#L168-L173)

  ```solidity
  function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256
  length)
      public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
      for (uint256 i; i < length;) {
          feeAmount += calculateFeeAmount(feeRate, amounts[i]);
          newAmounts[i] = amounts[i] - feeAmount; // 🚨 BUG: newAmounts has length 0
          unchecked { ++i; }
      }
  }

  Root Cause: The newAmounts return variable is automatically initialized as a zero-length array.
  Any assignment to newAmounts[i] causes array out-of-bounds access since the array has no
  allocated length.

  Function Failure Mechanism

  1. Function Call: User attempts to split or merge TVS
  2. Return Variable: newAmounts starts as zero-length array
  3. Assignment Attempt: Loop tries to assign newAmounts[0], newAmounts[1], etc.
  4. Out-of-Bounds Error: All assignments fail with "array out-of-bounds access" panic
  5. Complete Failure: Split/merge operations revert entirely

  Code References:
  - https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/Alignerz
  Vesting.sol#L1069 - splitTVS usage
  - https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/Alignerz
  Vesting.sol#L1013 - mergeTVS usage

  Impact

  Medium Severity: Core functionality completely broken but no direct fund loss

  - Complete Feature Failure: Split and merge operations never succeed
  - Core Functionality Broken: Major protocol features become unusable
  - User Experience Degradation: Users cannot modify their TVS configurations
  - Gas Waste: All split/merge attempts consume gas then revert completely

  Error Pattern:
  // Every split/merge operation fails with:
  // "panic: array out-of-bounds access (0x32)"
  splitTVS(projectId, [50, 50], nftId);      // ❌ REVERTS
  mergeTVS(projectId, mergedNftId, [projectId], [nftId1, nftId2]);     // ❌ REVERTS
  // 100% failure rate for these core functions

  Code Snippet

  Current vulnerable implementation:
  function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256
  length)
      public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
      // ❌ MISSING: newAmounts = new uint256[](length);

      for (uint256 i; i < length;) {
          feeAmount += calculateFeeAmount(feeRate, amounts[i]);
          newAmounts[i] = amounts[i] - feeAmount;  // Reverts: array length 0
          unchecked { ++i; }
      }
      // ❌ Never reached due to reverts above
  }

  Usage context showing impact:
  // splitTVS function in AlignerzVesting.sol
  (uint256 feeAmount, uint256[] memory newAmounts) =
  calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);
  allocation.amounts = newAmounts; // Never reached due to revert above

  Tool used

  Manual Review

  Recommendation

  Initialize array with proper length before assignment
  