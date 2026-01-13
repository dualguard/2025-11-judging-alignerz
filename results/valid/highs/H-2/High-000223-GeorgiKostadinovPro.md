# [000223] Any user calling `splitTVS` or `mergeTVS` will trigger a revert, permanently disabling TVS splitting and merging
  
  ### Summary

The lack of memory allocation for the `newAmounts` array in [`FeesManager.calculateFeeAndNewAmountForOneTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174) will cause a panic out-of-bounds revert for any execution of this function, causing a permanent denial-of-service for all users as any call into `splitTVS()` or `mergeTVS()` will revert when the function writes to the uninitialized array.

### Root Cause

In `FeesManager.sol: calculateFeeAndNewAmountForOneTVS`, the `newAmounts` array is never allocated before writing to it:
```js
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount; //@audit <--- newAmounts has length 0
    }
}
```

Because newAmounts is not initialized:
```js
newAmounts = new uint256[](length);
```

This function is used inside `AlignerzVesting.splitTVS()` and `AlignerzVesting.mergeTVS()`, making both user-facing features non-functional.

### Internal Pre-conditions

1. The vesting contract must hold a valid TVS allocation where `allocation.amounts.length > 0`.
2. A user must own a TVS NFT (normal operation).
3. The user calls `splitTVS()` or `mergeTVS()`.

### External Pre-conditions

None.
This vulnerability is entirely internal to the protocol and chain-independent.

### Attack Path

1. User calls AlignerzVesting.splitTVS(projectId, percentages, splitNftId) with valid inputs.
2. The function loads amounts = allocation.amounts, and sets nbOfFlows = amounts.length.
3. splitTVS() calls:
```js
(uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);
```
4. Inside the helper function, on the first loop iteration, the line:
```js
newAmounts[i] = ...
```
attempts to write to index 0 of a 0-length memory array.
5. EVM throws and the entire transaction reverts.

mergeTVS() fails identically.

### Impact

DoS of core functionality.

### PoC

A minimal reproduction using Remix:
Paste the following contract on remix:
```js
contract FeeManager {
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
            unchecked { ++i; }
        }
    }

    function calculateFeeAmount(uint256 feeRate, uint256 amount) public pure returns(uint256 feeAmount) {
        feeAmount = amount * feeRate / 10000;
    }
}
```


and call it with some example like:

```js
feeRate = 200
amounts = [100e6, 200e6, 300e6]
length = 3
```
it will revert with error: "RangeError: data out-of-bounds".

### Mitigation

Allocate the array before writing to it:
```js
uint256[] memory newAmounts = new uint256[](length);
```
  