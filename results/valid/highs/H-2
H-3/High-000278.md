# [000278] Uninitialized dynamic arrays cause related functions to revert
  
  ### Summary

Both `calculateFeeAndNewAmountForOneTVS()` and `_computeSplitArrays()` use dynamic memory arrays without initializing them. Any write to these arrays causes an array out-of-bounds panic, which makes functions like `splitTVS()` and `mergeTVS()` revert and effectively results in a DoS for these operations.

### Root Cause

In `calculateFeeAndNewAmountForOneTVS()` and `_computeSplitArrays()`, dynamic arrays are used in memory return values but never initialized before writing to them.

In `calculateFeeAndNewAmountForOneTVS()`, `uint256[] memory newAmounts` is one of the return values, but it is never initialized with a length. The function then writes to `newAmounts[i]`, which leads to an array out-of-bounds panic:

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174
```solidity
@>  function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

In `_computeSplitArrays()`, Allocation memory alloc is used as the return value, but its dynamic array fields (`amounts`, `vestingPeriods`, `vestingStartTimes`, `claimedSeconds`, `claimedFlows`) are never initialized with the needed length before being written to:

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113-L1141
```solidity
   function _computeSplitArrays(
        Allocation storage allocation,
        uint256 percentage,
        uint256 nbOfFlows
    )
        internal
        view
        returns (
 @>         Allocation memory alloc
        )
    {
		...
    }
```

In both functions, writing to uninitialized dynamic arrays in memory results in an array out-of-bounds panic and a revert.


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. A user calls `splitTVS()` or `mergeTVS()` on a TVS NFT, which internally routes into `calculateFeeAndNewAmountForOneTVS()` or `_computeSplitArrays()`.
2. When the code writes to the uninitialized dynamic arrays, it reverts due to array out-of-bounds, causing the operation to fail every time.

### Impact

All `splitTVS()` and `mergeTVS()` calls that trigger these uninitialized helpers will always fail and revert. As a result, users cannot split or merge their TVSs, causing a denial of service for these core vesting operations.

### PoC

Add the following test case to `test/AlignerzVestingProtocolTest.t.sol`:
```solidity
    function test_UninitializedArrayReverts() public {
        uint256 feeRate = 100;
        uint256 length = 2;
        uint256[] memory amounts = new uint256[](length);
        amounts[0] = 100e18;
        amounts[1] = 200e18;
        vesting.calculateFeeAndNewAmountForOneTVS(feeRate, amounts, length);
    }
```

Run with :
```solidity
forge test --mt test_UninitializedArrayReverts --force -vvvv
```

Outpot:
```solidity
...
  [17665] AlignerzVestingProtocolTest::test_UninitializedArrayReverts()
    ├─ [9355] ERC1967Proxy::fallback(100, [100000000000000000000 [1e20], 200000000000000000000 [2e20]], 2) [staticcall]
    │   ├─ [4093] AlignerzVesting::calculateFeeAndNewAmountForOneTVS(100, [100000000000000000000 [1e20], 200000000000000000000 [2e20]], 2) [delegatecall]
    │   │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    └─ ← [Revert] panic: array out-of-bounds access (0x32)

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 1.72s (62.33µs CPU time)

Ran 1 test suite in 1.74s (1.72s CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/AlignerzVestingProtocolTest.t.sol:AlignerzVestingProtocolTest
[FAIL: panic: array out-of-bounds access (0x32)] test_UninitializedArrayReverts() (gas: 17665)
```

This shows `calculateFeeAndNewAmountForOneTVS()` reverting due to an array out-of-bounds panic caused by writing to an uninitialized dynamic array.

### Mitigation

Both `calculateFeeAndNewAmountForOneTVS()` and `_computeSplitArrays()` should initialize their dynamic memory arrays before writing to them.

For `calculateFeeAndNewAmountForOneTVS()`:
```diff
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+		newAmounts = new uint256[](length);
		...
    }
```

For `_computeSplitArrays()`:
```diff
    function _computeSplitArrays(
        Allocation storage allocation,
        uint256 percentage,
        uint256 nbOfFlows
    )
        internal
        view
        returns (
            Allocation memory alloc
        )
    {
+       alloc.amounts           = new uint256[](nbOfFlows);
+       alloc.vestingPeriods    = new uint256[](nbOfFlows);
+       alloc.vestingStartTimes = new uint256[](nbOfFlows);
+       alloc.claimedSeconds    = new uint256[](nbOfFlows);
+       alloc.claimedFlows      = new bool[](nbOfFlows);
        ...
    }
```

  