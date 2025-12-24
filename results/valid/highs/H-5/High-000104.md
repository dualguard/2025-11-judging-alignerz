# [000104] Function `calculateFeeAndNewAmountForOneTVS` subtracts too much fees, giving the user less rewards
  
  ### Summary

Function [`calculateFeeAndNewAmountForOneTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169) subtracts `amounts[i] - feeAmount`, but `feeAmount` is the total fees added for each amount in the array, rewarding the user less when calling `mergeTVS` or `splitTVS`.
```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
        public
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
@>            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
``` 

### Root Cause

Function [calculateFeeAndNewAmountForOneTVS](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169) subtracts added fees from `amouns[i]`.

### Internal Pre-conditions

1. Array `uint256[] memory amounts` has more than 1 element.

### External Pre-conditions

None

### Attack Path

1. User calls `mergeTVS`.
2. User gets less rewards.

### Impact

User gets less rewards for each which gets increasingly worse for each iteration of the for loop:
```solidity
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
```
Instead of adding fee per `amounts[i]`, the fee subtracted is the `feeAmount` which is the total fees at the end of the for loop. 

### PoC

```solidity
    function test_PoC_TooMuchFeeSubtraction() public {
        uint256[] memory amounts = new uint256[](2);
        amounts[0] = 1000e18;
        amounts[1] = 1000e18;

        (uint256 feeAmount, uint256[] memory newAmounts) =
            vesting.calculateFeeAndNewAmountForOneTVS_correct(100, amounts, 2);

        console2.log("Fee amount subtracted is: ", feeAmount);
        console2.log("New amount of index 0: ", newAmounts[0]);
        console2.log("New amount of index 0: ", newAmounts[1]);

        assertNotEq(newAmounts[0], newAmounts[1]);
    }
```
Also add this function is the FeesManager since the actual `calculateFeeAndNewAmountForOneTVS` function is broken and reverts from other problems:
```solidity
    function calculateFeeAndNewAmountForOneTVS_correct(uint256 feeRate, uint256[] memory amounts, uint256 length)
        public
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
        newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
            unchecked {
                ++i;
            }
        }
    }
```

### Mitigation

```diff

    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
        public
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
+        uint256 fee = 0;
        for (uint256 i; i < length;) {
-           feeAmount += calculateFeeAmount(feeRate, amounts[i]);
-           newAmounts[i] = amounts[i] - feeAmount;
+           fee = calculateFeeAmount(feeRate, amounts[i]);
+           feeAmount += fee;
+           newAmounts[i] = amounts[i] - fee;
        }
```
  