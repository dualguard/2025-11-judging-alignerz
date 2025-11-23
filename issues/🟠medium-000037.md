# [000037] Loop counter is not incremented in `calculateFeeAndNewAmountForOneTVS` blocking merging and splitting TVS
  
  ### Summary

`calculateFeeAndNewAmountForOneTVS` is called when either merging or splitting TVS positions, which is used to cut the fees from the passed amounts. However, the loop counter is never incremented, and all calls to it will loop forever until it reverts with `arithmetic underflow or overflow`:
```solidity
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length; ) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
    }
}
```

### Root Cause

The loop counter is never incremented in `calculateFeeAndNewAmountForOneTVS`, https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L170.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

N/A

### Impact

Merging and splitting TVS can never be done.

### PoC

Apply the following diff before running the test (this fixes another issue that was reported separately):
```diff
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+   newAmounts = new uint256[](length);
    for (uint256 i; i < length; ) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
    }
}
```

```solidity
function test_NotIncrementingCounter() public {
    uint256[] memory amounts = new uint256[](2);
    amounts[0] = 1000;
    amounts[1] = 2000;

    vm.expectRevert(); // arithmetic underflow or overflow
    vesting.calculateFeeAndNewAmountForOneTVS(200, amounts, 2);
}
```

### Mitigation

Consider incrementing the loop counter at the very end of the iteration/loop:
```solidity
i++;
```
  