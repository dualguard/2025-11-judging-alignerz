# [000270] Incorrect fee calculation in `calculateFeeAndNewAmountForOneTVS()` leads to fund loss and accounting inconsistencies
  
  ### Summary

The fee calculation in `calculateFeeAndNewAmountForOneTVS()` is incorrect. It accumulates the fee across all flows and then subtracts this growing cumulative fee from each individual amount. This makes later flows pay more fees than earlier ones and over-deducts from TVS holders as a whole, while only sending the final cumulative fee to the treasury. As a result, user funds are lost and protocol accounting becomes inconsistent.

### Root Cause

In `calculateFeeAndNewAmountForOneTVS()`, the function loops over all entries in the amounts array, accumulates the fee in a single `feeAmount` variable, and uses this cumulative value to compute each `newAmounts[i]`:

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174
```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
@>          feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

Here, feeAmount is not the per-flow fee but the sum of all fees seen so far. This means the first element uses a small fee, the second element uses a larger cumulative fee, and so on. 

The later the index in the amounts array, the larger the deducted fee, because `newAmounts[i] = amounts[i] - feeAmount` uses the total accumulated fee instead of only the fee for that specific `amounts[i]`. At the end, the function returns only the final cumulative `feeAmount`.

This function is then used by both `mergeTVS()` and `splitTVS()` to compute `feeAmount` and updated amounts, and later send `feeAmount` tokens to the `treasury`.

In `mergeTVS()`:

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002-L1026
```solidity
@>      (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
        ...
@>      token.safeTransfer(treasury, feeAmount);
```

In `splitTVS()`:

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054-L1107
```solidity
@>      (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);
		...
@>      token.safeTransfer(treasury, feeAmount);
```

However, due to the feeAmount calculation flaw described above, while every element in `allocation.amounts[]` is reduced using the growing cumulative `feeAmount`, only the final cumulative value is sent to the treasury.

This means the total deduction from TVS holders’ allocations is greater than the amount credited to the treasury. The difference stays trapped in the contract and is effectively lost. This breaks both user balances and protocol accounting.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1.	Alice has a TVS allocation with `amounts = [100e18, 100e18, 100e18]` across three flows.
2.	Alice calls `splitTVS()` (or `mergeTVS()`), which internally calls `calculateFeeAndNewAmountForOneTVS()` with a fee rate of `10%` (`feeRate = 1000` in basis points).
3.	Inside `calculateFeeAndNewAmountForOneTVS()`, the cumulative `feeAmount` becomes `10e18`, then `20e18`, then `30e18`. The resulting `newAmounts` are `[90e18, 80e18, 70e18]`, so Alice’s total allocation is reduced from `300e18` to `240e18`, losing `60e18`.
4.	Only the final cumulative `feeAmount = 30e18` is returned and transferred to the treasury.
5.	The remaining `30e18` (`60e18` deducted from Alice minus `30e18` sent to treasury) is left stuck in the contract, creating an accounting mismatch and permanent user fund loss.

### Impact

This fee calculation issue has two major effects:
- For TVS holders, the longer the amounts array, the more later entries are over-charged, causing them to lose more tokens than the intended fee. 
- For the protocol, the total amount deducted from TVS allocations is greater than the amount actually transferred to the treasury, which breaks accounting and leaves part of the deducted funds locked inside the contract.

### PoC

Because `calculateFeeAndNewAmountForOneTVS()` has several other issues, we need to fix these bugs first to avoid immediate reverts and make this PoC executable.

First, replace `calculateFeeAndNewAmountForOneTVS()` in `FeesManager.sol` with the following code, which initializes the dynamic array `newAmounts` and correctly updates the loop index:
```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
            unchecked { ++i; }
        }
    }
```

Next, to demonstrate the scenario described in the **Attack Path** section, add this test to `test/AlignerzVestingProtocolTest.t.sol:
```solidity
    function test_CalculateFeeAndNewAmountFlaw() public {
        uint256 feeRate = 1000; // 10% in basis points
        uint256 length = 3;
		
        uint256[] memory amounts = new uint256[](length);
        amounts[0] = 100e18;
        amounts[1] = 100e18;
        amounts[2] = 100e18;
		// Call calculateFeeAndNewAmountForOneTVS() to calculate fee
        (
            uint256 feeAmount,
            uint256[] memory newAmounts
        ) = vesting.calculateFeeAndNewAmountForOneTVS(feeRate, amounts, length);

        // Cumulative fee: 10 + 20 + 30 = 30e18 returned
        assertEq(feeAmount, 30e18, "unexpected cumulative fee");

        // Each amount is reduced by the growing cumulative fee
        assertEq(newAmounts[0], 90e18, "first amount should be 90e18");
        assertEq(newAmounts[1], 80e18, "second amount should be 80e18");
        assertEq(newAmounts[2], 70e18, "third amount should be 70e18");

        // Total user allocation drops by 60e18, but only 30e18 is sent as fee
        uint256 originalTotal = amounts[0] + amounts[1] + amounts[2];
        uint256 newTotal = newAmounts[0] + newAmounts[1] + newAmounts[2];

        assertEq(originalTotal, 300e18, "original total should be 300e18");
        assertEq(newTotal, 240e18, "new total should be 240e18");
        assertEq(originalTotal - newTotal, 60e18, "user loses 60e18 in total");
    }
```

And run with:
```shell
forge test --mt test_CalculateFeeAndNewAmountFlaw  --force -vvvv
```

Output:
```shell
[PASS] test_CalculateFeeAndNewAmountFlaw() (gas: 48156)
Traces:
  [48156] AlignerzVestingProtocolTest::test_CalculateFeeAndNewAmountFlaw()
    ├─ [14206] ERC1967Proxy::fallback(1000, [100000000000000000000 [1e20], 100000000000000000000 [1e20], 100000000000000000000 [1e20]], 3) [staticcall]
    │   ├─ [8927] AlignerzVesting::calculateFeeAndNewAmountForOneTVS(1000, [100000000000000000000 [1e20], 100000000000000000000 [1e20], 100000000000000000000 [1e20]], 3) [delegatecall]
    │   │   └─ ← [Return] 30000000000000000000 [3e19], [90000000000000000000 [9e19], 80000000000000000000 [8e19], 70000000000000000000 [7e19]]
    │   └─ ← [Return] 30000000000000000000 [3e19], [90000000000000000000 [9e19], 80000000000000000000 [8e19], 70000000000000000000 [7e19]]
    ├─ [0] VM::assertEq(30000000000000000000 [3e19], 30000000000000000000 [3e19], "unexpected cumulative fee") [staticcall]
    │   └─ ← [Return]
    ├─ [0] VM::assertEq(90000000000000000000 [9e19], 90000000000000000000 [9e19], "first amount should be 90e18") [staticcall]
    │   └─ ← [Return]
    ├─ [0] VM::assertEq(80000000000000000000 [8e19], 80000000000000000000 [8e19], "second amount should be 80e18") [staticcall]
    │   └─ ← [Return]
    ├─ [0] VM::assertEq(70000000000000000000 [7e19], 70000000000000000000 [7e19], "third amount should be 70e18") [staticcall]
    │   └─ ← [Return]
    ├─ [0] VM::assertEq(300000000000000000000 [3e20], 300000000000000000000 [3e20], "original total should be 300e18") [staticcall]
    │   └─ ← [Return]
    ├─ [0] VM::assertEq(240000000000000000000 [2.4e20], 240000000000000000000 [2.4e20], "new total should be 240e18") [staticcall]
    │   └─ ← [Return]
    ├─ [0] VM::assertEq(60000000000000000000 [6e19], 60000000000000000000 [6e19], "user loses 60e18 in total") [staticcall]
    │   └─ ← [Return]
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.79s (137.08µs CPU time)
```


### Mitigation

Update c`alculateFeeAndNewAmountForOneTVS()` so that each flow’s fee is computed and applied independently, and `feeAmount` as the total fee equals the sum of all per-flow fees. Building on the fixes for the other issues, the fee calculation mechanism can be corrected as follows:
```diff
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+       uint256 fee;
	    newAmounts = new uint256[](length);        
        for (uint256 i; i < length;) {
-           feeAmount += calculateFeeAmount(feeRate, amounts[i]);
-           newAmounts[i] = amounts[i] - feeAmount;
+           fee = calculateFeeAmount(feeRate, amounts[i]);
+           newAmounts[i] = amounts[i] - fee;
+			feeAmount += fee;
			unchecked { ++i; }
        }
    }
```

  