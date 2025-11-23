# [000291] calculateFeeAndNewAmountForOneTVS() Will Always Revert Due To a Memory Array That Never Gets a Specific Size Causes mergeTVS() and splitTVS() to Never Execute
  
  ### Summary

As stated in solodity's docs https://docs.soliditylang.org/en/latest/types.html#allocating-memory-arrays
You can't resize or define an element of a `memory array` without pre-assigning a Its size, but in `calculateFeeAndNewAmountForOneTVS()` in `FeeManager.sol` tries to access an element in `newAmount[]` (that is null) causing a revert.

### Root Cause

Trying to write to `newAmount[i]` without initializing the size of the memory array
https://github.com/dualguard/2025-11-alignerz-DunateIIo/blob/fe542df71d42e3a855f2b014032440ccc2b40da4/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

N/A 

### Impact

mergeTVS() and splitTVS() which are major functions, will NEVER work.

### PoC

_No response_

### Mitigation

This is a complete correct function (other bugs are reported in separate reports)

```
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 totalFeeAmount, uint256[] memory newAmounts) {
    // Fix 1: Initialize the array
    newAmounts = new uint256[](length);

    for (uint256 i; i < length;) {
        uint256 currentFee = calculateFeeAmount(feeRate, amounts[i]);
        // FIX 2: Subtract only the current fee, not accumulative.
        newAmounts[i] = amounts[i] - currentFee;
        totalFeeAmount += currentFee;

        // FIX 3: Increment the counter
        unchecked { ++i; }
    }
}
```
  