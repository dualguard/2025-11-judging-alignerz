# [000054] Memory Not Allocated in `calculateFeeAndNewAmountForOneTVS`
  
  ### Summary

In the `FeesManager.sol, Lines 169–174` The `newAmounts` array is declared but not allocated in memory. In Solidity, memory arrays must be explicitly created using `new uint256[](length)`. Writing to an unallocated array causes a revert (Panic 0x32: out-of-bounds memory access).

### Root Cause

```solidity
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;                  // @<<< newAmounts not allocated
    }
}
```
    
https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169



### Impact

- Any function calling `calculateFeeAndNewAmountForOneTVS` will revert.
    
- Operations like splitting or merging TVS tokens relying on this function will fail.
    
    

### PoC


**Run this contract using the Remix IDE**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.29;

contract MemoryBugExample {
    uint256 constant BASIS_POINT = 10_000;

    function calculateFeeAmount(uint256 feeRate, uint256 amount) public pure returns (uint256) {
        return amount * feeRate / BASIS_POINT;
    }

    // BUGGY FUNCTION: newAmounts not allocated
    function buggyCalculate(uint256 feeRate, uint256[] memory amounts) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        uint256 length = amounts.length;
    
        //  newAmounts is declared in returns but not allocated
        for (uint256 i = 0; i < length; ++i) {
            uint256 itemFee = calculateFeeAmount(feeRate, amounts[i]);
            feeAmount += itemFee;
            newAmounts[i] = amounts[i] - itemFee; // REVERTS here   <----
        }
    }



    // Second Example
     function buggy() public pure returns (uint256[] memory arr) {
    arr[0] = 10; //  WRONG
}

    function notBuggy() public pure returns (uint256[] memory arr){
        arr = new uint256[](5);
        arr[0] = 10;
    }
}

```

### Mitigation

Allocate memory for `newAmounts` before the loop:
    

```diff
+ newAmounts = new uint256[](length);
```


  