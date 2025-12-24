# [000148] Broken and unsafe calculateFeeAndNewAmountForOneTVS implementation
  
  The return array inside calculateFeeAndNewAmountForOneTVS  (newAmounts) is never allocated (no new uint256[](length)), so writing to newAmounts[i] will revert, causing this function to never work. The loop also never increments i (no ++i or unchecked { ++i; }) — that makes it an infinite loop that won't ever finish executing.

The code also subtracts feeAmount (the cumulative fee so far) from amounts[i] instead of subtracting the fee for the current flow; that produces incorrect newAmounts (and incorrect fee accounting) because feeAmount is being used both as the running total of fees and as the value subtracted from each individual flow, later flows end up having earlier flows' fees deducted from them.

The function as written is non-functional and will revert on all calls.
  