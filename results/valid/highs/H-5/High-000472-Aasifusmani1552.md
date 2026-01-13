# [000472] The `calculateFeeAndNewAmountForOneTVS` function in `FeesManager` contract incorrectly deducts cumulative `feeAmount` from every flow instead of deducting the fee from that specific flow
  
  ### Summary

The `calculateFeeAndNewAmountForOneTVS` function incorrectly subtracts the cumulative `feeAmount` from each element of `amounts` (each flow), instead of subtracting only the fee for that specific element (flow). This logic produces increasingly larger amount of fee with every iteration for the next element (flow) which is not intended at all.

Now, Because `feeAmount` grows with every iteration, later iterations may attempt to compute:-
```solidity
newAmounts[i] = amounts[i] - cumulativeFee
```
If the cumulative fee becomes larger than `amounts[i]` for any index, **this results in an underflow, which causes a hard revert in Solidity ≥0.8 due to checked arithmetic.**

And because both `mergeTVS` and `splitTVS` use this function, both functionalities return incorrect values and deduct too much fee from user or revert entirely because of underflow.

### Root Cause

The code subtracts the continuously growing cumulative fee `feeAmount` rather than the fee for the current element:
```solidity
function calculateFeeAndNewAmountForOneTVS(...) {
    for (uint256 i; i < length; ) {
        uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
@>      feeAmount += fee;                     // cumulative fee
 @>     newAmounts[i] = amounts[i] - feeAmount;
    }
}
```

Here is the link to the affected code:-
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169

### Internal Pre-conditions

This doesn't require any pre-conditions.

### External Pre-conditions

This doesn't require any pre-conditions.

### Attack Path

1. User calls `mergeTVS` or `splitTVS` function.
2. `mergeTVS` or `splitTVS` calls `calculateFeeAndNewAmountForOneTVS` function internally to calculate the fee (either for merge or split).
3. The `calculateFeeAndNewAmountForOneTVS` function either reverts with **Underflow** or deducts very large amount of fee from flow after flow.

### Impact

1. **Incorrect Business Logic** – Each element is reduced by the `total accumulated fee` (feeAmount) instead of the fee for that specific item, breaking the expected fee-calculation model.
2. **Underflow Reverts Risk**– When cumulative fees exceed any `amounts[i]`, the subtraction causes a Solidity arithmetic underflow which causes guaranteed revert.
3. **DoS on Vesting Operations** – `mergeTVS` and `splitTVS` cannot proceed correctly if fee deduction reverts due to underflow.
4. **Incorrect Fee Distribution** – Later elements get drastically reduced or zeroed amounts.

**Severity - HIGH**

### PoC

- Please copy/paste the given test in `AlignerzVestingProtocolTest.t.sol` test file to run the PoC
- Run the test with the following command:- `forge test --mt test_underflowInfeeCalculationPoC -vvvv`
- I have modified the `calculateFeeAndNewAmountForOneTVS` function to remove other bugs present in the function (OOB array acess and counter incrementer bugs). Please update the function as follows:-
```diff
   function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
        public
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
+        require(amounts.length == length);
+      newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
+            ++i;
        }
    }
```

Here is the PoC:-
```solidity
    function test_underflowInfeeCalculationPoC() public {
        uint256 feeRate = 200;
        uint256[] memory amounts = new uint256[](5);
        amounts[0] = 100;
        amounts[1] = 200; //increasing amounts in later flows to show the underflow
        amounts[2] = 100;
        amounts[3] = 200;
        amounts[4] = 10;

        //directly calling the function to show the issue
        vm.expectRevert();
        vesting.calculateFeeAndNewAmountForOneTVS(feeRate, amounts, amounts.length);
    }
```

**Logs:-**
```solidity
Ran 1 test for test/AlignerzVestingProtocolTest.t.sol:AlignerzVestingProtocolTest
[PASS] test_underflowInfeeCalculationPoC() (gas: 33205)
Traces:
  [33205] AlignerzVestingProtocolTest::test_underflowInfeeCalculationPoC()
    ├─ [0] VM::expectRevert(custom error 0xf4844814)
    │   └─ ← [Return]
    ├─ [16342] ERC1967Proxy::fallback(200, [100, 200, 100, 200, 10], 5) [staticcall]
    │   ├─ [11062] AlignerzVesting::calculateFeeAndNewAmountForOneTVS(200, [100, 200, 100, 200, 10], 5) [delegatecall]
    │   │   └─ ← [Revert] panic: arithmetic underflow or overflow (0x11)
    │   └─ ← [Revert] panic: arithmetic underflow or overflow (0x11)
    └─ ← [Return]
```

### Mitigation

To mitigate the issue, only deduct the per flow calculated fee not the cumulative fee:-
```diff
    function calculateFeeAndNewAmountForOneTVS(...)
        public
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
      //-----rest of the logic
        for (...) {
+            uint256 perFee = calculateFeeAmount(feeRate, amounts[i]);
+            feeAmount += perFee;
+           newAmounts[i] = amounts[i] - perFee;
        }
    }
// ...existing code...
```
  