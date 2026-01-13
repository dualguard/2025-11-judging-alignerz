# [000448] calculateFeeAndNewAmountForOneTVS will always revert
  
  ### Summary

Merge and Split will be DoS because the call to `calculateFeeAndNewAmountForOneTVS` can never succeed. This is because the returned array `newAmounts` is never initialized to the length of `amounts`. Thus, when trying to run the line `newAmounts[i] = amounts[i] - feeAmount;`, the transaction will revert with an array out-of-bounds access error.

### Root Cause

Return array not initialized in `calculateFeeAndNewAmountForOneTVS`.

### Attack Path

A user tries to split or merge their NFT.

The call will never succeed.

### Impact

Permanent DoS of `splitTVS` and `mergeTVS`.

### PoC

Copy this function in the testfile. It will revert with `panic: array out-of-bounds access (0x32)`

```solidity
    function test_calcFees() public {
        uint256 feeRate;
        uint256[] memory amounts = new uint[](1);

        (uint256 fee, uint[] memory newAmounts) = vesting.calculateFeeAndNewAmountForOneTVS(feeRate, amounts, amounts.length);
    }
```

### Mitigation

Initialize the returned array.

```diff
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+       newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```
  