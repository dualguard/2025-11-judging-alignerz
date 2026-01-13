# [000279] `calculateFeeAndNewAmountForOneTVS()` missing index increment leads to split or merge TVS DoS
  
  ### Summary

`calculateFeeAndNewAmountForOneTVS()` lacks the index increment inside the for loop, causing both `mergeTVS()` and `splitTVS()` to revert and resulting in a DoS.

### Root Cause

In `calculateFeeAndNewAmountForOneTVS()`, a for loop is used to iterate over the amounts array, but the loop index `i` is never incremented. 

As a result, the loop never finishes normally, running until it hits an out-of-gas condition.

Another case is when feeRate is not zero and `feeAmount` keeps accumulating, causing `amounts[i] - feeAmount` to underflow and revert.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174
```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```
In both situations, the helper becomes unusable and any function that relies on it will also revert.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1.	Any user who owns a TVS NFT calls `mergeTVS()` or `splitTVS()` on the vesting contract for a project where the TVS has at least one flow.
2.	These operations will revert due to `calculateFeeAndNewAmountForOneTVS()` running out of gas or triggering an arithmetic panic.


### Impact

All `mergeTVS()` and `splitTVS()` calls that depend on `calculateFeeAndNewAmountForOneTVS()` will consistently revert and result in a DoS.

### PoC


Add the following test case to `test/AlignerzVestingProtocolTest.t.sol`:
```solidity
    function test_FeeCalculationDoS() public {
        uint256 feeRate = 0;
        uint256 length = 2;
        uint256[] memory amounts = new uint256[](length);
        amounts[0] = 100e18;
        amounts[1] = 200e18;
        // [FAIL: panic: array out-of-bounds access (0x32)]
        vesting.calculateFeeAndNewAmountForOneTVS(feeRate, amounts, length);
    }
```

Run with:
```shell
forge test --mt test_FeeCalculationDoS  --force -vvvv
```

Output:
```shell
	...
  [1040429577] AlignerzVestingProtocolTest::test_FeeCalculationDoS()
    ├─ [1040421230] ERC1967Proxy::fallback(0, [100000000000000000000 [1e20], 200000000000000000000 [2e20]], 2) [staticcall]
    │   ├─ [1040415974] AlignerzVesting::calculateFeeAndNewAmountForOneTVS(0, [100000000000000000000 [1e20], 200000000000000000000 [2e20]], 2) [delegatecall]
    │   │   └─ ← [OutOfGas] EvmError: OutOfGas
    │   └─ ← [Revert] EvmError: Revert
    └─ ← [Revert] EvmError: Revert
```


### Mitigation

Add an index increment statement inside the loop in `calculateFeeAndNewAmountForOneTVS()`:

```diff
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        ...
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
+           unchecked { ++i; }
        }
    }
```
  