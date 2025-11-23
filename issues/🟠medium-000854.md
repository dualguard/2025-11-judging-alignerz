# [000854] Array Initialization Error in Split Function Breaks TVS Operations
  
    ## Summary

  The `_computeSplitArrays` function in `AlignerzVesting.sol` attempts to assign values to zero-length dynamic arrays, causing transaction reverts and making all TVS split operations completely unusable.

  ## Vulnerability Detail

  The vulnerability exists in the split computation logic where memory struct arrays are not initialized with proper length before assignment:

  **Location**: [`AlignerzVesting.sol:1131-1140`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1131-L1140)

  ```solidity
  function _computeSplitArrays(
      Allocation storage allocation,
      uint256 percentage,
      uint256 nbOfFlows
  ) internal view returns (Allocation memory alloc) {
      uint256[] memory baseAmounts = allocation.amounts;
      uint256[] memory baseVestings = allocation.vestingPeriods;
      uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
      uint256[] memory baseClaimed = allocation.claimedSeconds;
      bool[] memory baseClaimedFlows = allocation.claimedFlows;
      alloc.assignedPoolId = allocation.assignedPoolId;
      alloc.token = allocation.token;
      for (uint256 j; j < nbOfFlows;) {
          alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT; // 🚨 BUG: Array has length 0
          alloc.vestingPeriods[j] = baseVestings[j]; // 🚨 BUG: Array has length 0
          alloc.vestingStartTimes[j] = baseVestingStartTimes[j]; // 🚨 BUG: Array has length 0
          alloc.claimedSeconds[j] = baseClaimed[j]; // 🚨 BUG: Array has length 0
          alloc.claimedFlows[j] = baseClaimedFlows[j]; // 🚨 BUG: Array has length 0
          unchecked {
              ++j;
          }
      }
  }

  Root Cause: The alloc variable is a memory struct where dynamic arrays start as zero-length arrays. Attempting to assign to alloc.amounts[j] when alloc.amounts.length == 0 causes array out-of-bounds
  access.

  Function Failure Mechanism

  1. TVS Split Request: User calls splitTVS function
  2. Array Creation: alloc struct created with zero-length arrays
  3. Assignment Attempt: Loop tries to assign alloc.amounts[0], alloc.vestingPeriods[0], etc.
  4. Out-of-Bounds Error: All assignments fail with "array out-of-bounds access" panic
  5. Complete Failure: Split operation reverts entirely, no partial execution

  Function Usage Context:
  // splitTVS function calls _computeSplitArrays - will revert
  Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);

  Impact

  Medium Severity: Core functionality completely broken but no direct fund loss

  - Split Function Unusable: All TVS splitting attempts fail with array out-of-bounds errors
  - Core Feature Broken: Key functionality promised in protocol documentation doesn't work
  - User Experience: Users cannot split their TVS for liquidity management or portfolio rebalancing
  - Gas Waste: All split attempts consume gas then revert completely

  Error Pattern:
  // Every split operation fails with:
  // "panic: array out-of-bounds access (0x32)"
  splitTVS(projectId, [50, 50], nftId);      // ❌ REVERTS
  // 100% failure rate for split operations

  Code Snippet

  Current vulnerable implementation:
  function _computeSplitArrays(
      Allocation storage allocation,
      uint256 percentage,
      uint256 nbOfFlows
  ) internal view returns (Allocation memory alloc) {
      // ... variable assignments ...
      // ❌ MISSING: Array initialization
      for (uint256 j; j < nbOfFlows;) {
          alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT; // Reverts: array length 0
          alloc.vestingPeriods[j] = baseVestings[j];                      // Reverts: array length 0
          alloc.vestingStartTimes[j] = baseVestingStartTimes[j];          // Reverts: array length 0
          alloc.claimedSeconds[j] = baseClaimed[j];                       // Reverts: array length 0
          alloc.claimedFlows[j] = baseClaimedFlows[j];                    // Reverts: array length 0
          unchecked {
              ++j;
          }
      }
      // ❌ Never reached due to reverts above
  }

  Tool used

  Manual Review
  