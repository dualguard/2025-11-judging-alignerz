# [001042] Users will lose exponentially increasing funds due to cumulative fee calculation bug
  
  ## Summary

The cumulative fee calculation in `FeesManager.calculateFeeAndNewAmountForOneTVS()` will cause exponential fund loss for users as they merge or split TVS allocations, with losses increasing proportionally to the number of flows (users can lose 100% extra with 3 flows, up to 2000%+ extra with 50 flows).

## Root Cause
https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L175
In `FeesManager.sol:172` the fee deduction uses cumulative `feeAmount` instead of individual fee per flow, causing each subsequent flow to have progressively larger fee deducted.

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
    public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    // newAmounts = new uint256[](length);  // Assume this is fixed as reported in my other finding
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);  // Accumulates total fee
        newAmounts[i] = amounts[i] - feeAmount;  // @audit-issue Uses cumulative fee instead of individual
        unchecked {
            ++i; // Assume this is fixed as reported in my other finding also
        }
    }
}
```

The bug is on line 172: `newAmounts[i] = amounts[i] - feeAmount`

- `feeAmount` accumulates across all iterations: 20, 40, 60, 80...
- Each flow subtracts the **cumulative** fee instead of its **individual** fee
- This creates exponentially increasing losses

**Correct behavior**: Each flow should subtract only its own individual fee (constant 20 for each 1000 tokens at 2% rate)

**Buggy behavior**: Each flow subtracts the sum of all previous fees plus its own fee

## Internal Pre-conditions

1. User needs to have TVS allocations with multiple flows (created through merging TVS NFTs)
2. Protocol owner needs to set merge or split fee rate to any non-zero value (e.g., 2% = 200 basis points)
3. User calls `mergeTVS()` or `splitTVS()` on multi-flow allocations

## External Pre-conditions

None - this is a pure mathematical logic error in fee calculation.

## Attack Path

This is a vulnerability path (not an attack), triggered during normal protocol operations:

1. **User creates 3 separate TVS NFTs through reward claims**, each with 1 flow of 1000 tokens
2. **User calls `mergeTVS()` to consolidate into 1 NFT**, paying 2% merge fee
3. **Contract calls `calculateFeeAndNewAmountForOneTVS(200, [1000, 1000, 1000], 3)`** at AlignerzVesting.sol:1013
4. **Buggy calculation executes**:
   - Iteration 0: `feeAmount = 20`, `newAmounts[0] = 1000 - 20 = 980` ✓
   - Iteration 1: `feeAmount = 40`, `newAmounts[1] = 1000 - 40 = 960` ❌
   - Iteration 2: `feeAmount = 60`, `newAmounts[2] = 1000 - 60 = 940` ❌
5. **User's allocation becomes [980, 960, 940] = 2880 tokens** instead of expected 2940 tokens
6. **User lost 120 tokens (4% total) instead of 60 tokens (2% fee)**

## Impact

Users suffer exponentially increasing losses proportional to the square of the number of flows in their TVS allocation. With 3 flows at 2% fee rate, users lose an extra 2% (total 4%). With 10 flows, users lose an extra 9% (total 11%). With 50 flows, users could lose over 50% of their tokens.


### Loss Calculation Table:

With 2% fee rate (200 basis points) and N flows of 1000 tokens each:

| Flows | Expected Fee | Actual Loss | Extra Loss | Extra Loss % | Total Loss % |
|-------|--------------|-------------|------------|--------------|--------------|
| 1     | 20 tokens    | 20 tokens   | 0 tokens   | 0%           | 2%           |
| 2     | 40 tokens    | 60 tokens   | 20 tokens  | 50%          | 3%           |
| 3     | 60 tokens    | 120 tokens  | 60 tokens  | 100%         | 4%           |
| 5     | 100 tokens   | 300 tokens  | 200 tokens | 200%         | 6%           |
| 10    | 200 tokens   | 1,100 tokens| 900 tokens | 450%         | 11%          |
| 20    | 400 tokens   | 4,200 tokens| 3,800 tokens| 950%        | 21%          |
| 50    | 1,000 tokens | 25,500 tokens| 24,500 tokens| 2,450%    | 51%          |

This affects:
- **`mergeTVS()` at AlignerzVesting.sol:1013**: Called when consolidating multiple TVS NFTs
- **`splitTVS()` at AlignerzVesting.sol:1069**: Called when dividing a TVS NFT
- **All users** who perform these operations on multi-flow allocations

**Note:** Currently this bug cannot manifest because the uninitialized array bug (separate issue) and not incrementing the index counter causes the function to revert before reaching the fee calculation logic. However, once those bugs are fixed, this cumulative fee bug will immediately cause user fund loss.

## PoC

_Prerequisite: The uninitialized array (`newAmounts`) and index counter increment bug must be fixed first for this scenario to execute.

- Add this test to `test/AlignerzVestingProtocolTest.t.sol`:test_C2_CumulativeFeeBug`


```solidity
   function test_C2_CumulativeFeeBug() public {
        vm.startPrank(projectCreator);
        vesting.setMergeFeeRate(200); // 2% fee

        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 1000 ether;
        amounts[1] = 1000 ether;
        amounts[2] = 1000 ether;

        // This function is public so we can call it directly for testing purposes
        (uint256 feeAmount, uint256[] memory newAmounts) = vesting.calculateFeeAndNewAmountForOneTVS(
            vesting.mergeFeeRate(),
            amounts,
            amounts.length
        );

        // Expected fee breakdown:
        // Flow 1: 1000 * 2% = 20
        // Flow 2: 1000 * 2% = 20
        // Flow 3: 1000 * 2% = 20
        // Total Fee: 60

        // Correct new amounts should be:
        // newAmounts[0] = 1000 - 20 = 980
        // newAmounts[1] = 1000 - 20 = 980
        // newAmounts[2] = 1000 - 20 = 980

        // Buggy new amounts will be:
        // newAmounts[0] = 1000 - 20 = 980
        // newAmounts[1] = 1000 - (20 + 20) = 960
        // newAmounts[2] = 1000 - (20 + 20 + 20) = 940

        assertEq(feeAmount, 60 ether, "Total fee should be 60");
        assertEq(newAmounts[0], 980 ether, "newAmounts[0] should be 980");
        assertEq(newAmounts[1], 960 ether, "newAmounts[1] should be 960 (bug)");
        assertEq(newAmounts[2], 940 ether, "newAmounts[2] should be 940 (bug)");
    }
```

## Mitigation

Fix the fee calculation in `FeesManager.sol:172` to use individual fee instead of cumulative:

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
    public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    newAmounts = new uint256[](length);
    for (uint256 i; i < length;) {
        uint256 individualFee = calculateFeeAmount(feeRate, amounts[i]);  // ✓ Calculate per-flow fee
        feeAmount += individualFee;  // ✓ Accumulate total fee for treasury transfer
        newAmounts[i] = amounts[i] - individualFee;  // ✓ Deduct only individual fee, not cumulative
        unchecked {
            ++i;
        }
    }
}
```

This ensures:
1. Each flow is charged exactly its proportional fee (e.g., 2% of its amount)
2. Total fee accumulated in `feeAmount` is still correct for treasury transfer
3. Users receive the correct remaining amount after paying the advertised fee rate

  