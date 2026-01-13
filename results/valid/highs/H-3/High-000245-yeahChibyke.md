# [000245] Uninitialized Memory Arrays in `AlignerzVesting::_computeSplitArrays()` Causes Complete Failure of NFT Split Functionality
  
  ### Summary

The `::_computeSplitArrays()` function attempts to write to uninitialized dynamic arrays in memory, causing all calls to `AlignerzVesting::splitTVS()` to revert. This completely breaks the NFT splitting feature, preventing users from dividing their vesting positions into multiple smaller positions. Any user attempting to split their NFT will experience a transaction failure.

### Root Cause

The function declares a return variable `Allocation memory alloc` but never initializes its dynamic array fields before attempting to write to them:

```solidity
   function _computeSplitArrays(Allocation storage allocation, uint256 percentage, uint256 nbOfFlows)
      internal
      view
      returns (Allocation memory alloc)  // ← alloc is declared but arrays are uninitialized
   {
      uint256[] memory baseAmounts = allocation.amounts;
      uint256[] memory baseVestings = allocation.vestingPeriods;
      uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
      uint256[] memory baseClaimed = allocation.claimedSeconds;
      bool[] memory baseClaimedFlows = allocation.claimedFlows;
      alloc.assignedPoolId = allocation.assignedPoolId;
      alloc.token = allocation.token;
      
      for (uint256 j; j < nbOfFlows;) {
         alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;  //  REVERT!
         alloc.vestingPeriods[j] = baseVestings[j];                       //  Array length is 0
         alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
         alloc.claimedSeconds[j] = baseClaimed[j];
         alloc.claimedFlows[j] = baseClaimedFlows[j];
         unchecked {
               ++j;
         }
      }
   }
```

### Internal Pre-conditions

1. User must own an NFT representing a vesting position (from either a bidding or reward project)
2. The NFT's allocation must have at least one token flow (i.e., `allocation.amounts.length > 0`)
3. User must call splitTVS() with valid parameters:
   - `projectId`: Valid project ID
   - `percentages`: Array of percentages that sum to 10,000 basis points
   - `splitNftId`: ID of the NFT they own

### External Pre-conditions

None

### Attack Path

**Scenario:**

1. Alice receives vesting NFT-1 which has 10,000 tokens vesting over 365 days
2. She wants to sell 40% while keeping 60%
3. She calls `::splitTVS(0)`:

   ```solidity
   vesting.splitTVS(
      projectId: 0,
      percentages: [6000, 4000],  // 60% and 40%
      splitNftId: 1
   )
   ```

4. Transaction execution flow:

   ```
   splitTVS()
   ↓
   Retrieves allocation from storage (contains 1 flow)
   ↓
   Sets nbOfFlows = 1
   ↓
   Calls _computeSplitArrays(allocation, 6000, 1) for first percentage
   ↓
   Inside _computeSplitArrays:
   - Creates empty Allocation memory alloc
   - alloc.amounts.length = 0  ← Problem starts here
   - alloc.vestingPeriods.length = 0
   - Loop tries: alloc.amounts[0] = ... ← REVERT!
   ```

5. Transaction reverts
6. Alice cannot split her NFT

### Impact

100% failure rate for all split attempts.

### PoC

_No response_

### Mitigation

Initialize all dynamic arrays in the returned struct before writing to them.
  