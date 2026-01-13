# [000474] [DoS] `calculateFeeAndNewAmountForOneTVS` hard reverts due to out-of-bounds write on unallocated `newAmounts` array
  
  ### Summary

The `calculateFeeAndNewAmountForOneTVS` function in the `FeeManager` contract (inherited by the vesting contract) is intended to return a cumulative `feeAmount` and an array of `newAmounts` after deducting the fee for each element (from amounts array).
However, the function never allocates memory for `newAmounts` and immediately attempts to write to `newAmounts[i]`.
This results in **Out-of-Bounds Array Access (panic 0x32)** and causes the function to always revert.

Since this function is used by both `mergeTVS` and `splitTVS`, it completely breaks both functionalities.

### Root Cause

The root cause of the issue is not allocating memory to the array before accessing it.

Here:-
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

Here is the link to affected code:-
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169

### Internal Pre-conditions

It doesn't require any pre-conditions.

### External Pre-conditions

It doesn't require any pre-conditions(wheter internal or external).

### Attack Path

1. Some user calls `mergeTVS` or `splitTVS` function.
2. `mergeTVS` or `splitTVS` calls `calculateFeeAndNewAmountForOneTVS` function to calculate the fee.
3. The `calculateFeeAndNewAmountForOneTVS` function reverts because of accessing array without allocating memory.

### Impact

1. **Completely Breaks Merge and Split TVS functionalities** – Because the merge and split TVS functions depend on this function for the calculation of fee, This completely halts those two functionalities as well.
2. **DoS (Denial of Service)** – Out-of-bounds write triggers a **panic(0x32)** causing the function to always revert, making it impossible for callers to complete execution.
3. **Non-Functional Logic** – The function cannot return `newAmounts` because the array is never allocated, making the entire fee-calculation flow unusable.
4. **Memory Safety Violation** – Attempting to write outside the array boundaries violates Solidity’s memory model and halts execution.

**Severity:- High** 

### PoC

- Copy paste the given test in `AlignerzVestingProtocolTest.t.sol` file.
- Run with command `forge test --mt test_OutofBoundArrayAccess -vvvv`

```solidity
function test_OutofBoundArrayAccess() public {
        uint256 feeRate = 200;
        uint256[] memory amounts = new uint256[](5);
        amounts[0] = 10;
        amounts[1] = 10;
        amounts[2] = 10;
        amounts[3] = 10;
        amounts[4] = 10;

        //directly calling the function to show the issue
        vesting.calculateFeeAndNewAmountForOneTVS(feeRate, amounts, amounts.length);
    }
```

Logs:-
```solidity
[19853] AlignerzVestingProtocolTest::test_OutofBoundArrayAccess()
    ├─ [9910] ERC1967Proxy::fallback(200, [10, 10, 10, 10, 10], 5) [staticcall]
    │   ├─ [4630] AlignerzVesting::calculateFeeAndNewAmountForOneTVS(200, [10, 10, 10, 10, 10], 5) [delegatecall]
    │   │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    └─ ← [Revert] panic: array out-of-bounds access (0x32)

```

### Mitigation

To mitigate this issue, allocate memory to `newAmounts` array before attempting to write into it, like:-
```diff
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    require(length == amounts.length, "length != amounts.length");

++    newAmounts = new uint256[](length);
    // ---- ------ Rest of the logic -------
}

```
  