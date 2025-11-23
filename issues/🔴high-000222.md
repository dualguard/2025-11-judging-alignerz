# [000222] Incorrect cumulative fee calculation subtracts excessive amounts from user vesting flows, causing token loss and incorrect TVS allocations
  
  ### Summary

The incorrect fee calculation in [`FeesManager.calculateFeeAndNewAmountForOneTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174) will cause excessive token subtraction from user vesting flows, resulting in silent loss of vested tokens for users as the function cumulatively subtracts all previous fees from each flow instead of only its own fee. This will corrupt TVS allocation math during both `splitTVS()` and `mergeTVS()`, causing users to lose part of their vesting amounts.

### Root Cause
When we split, the protocol takes 0.5% fee from each portion as per whitepaper:
```js
A 12-month vesting TVS holding 1,200 $A26Z tokens (100/month) is split into 70%, 20%, and
10% portions (splitting fee 0.5%):
• TVS 1 (70%) → 69.65 $A26Z/month
• TVS 2 (20%) → 19.9 $A26Z/month
• TVS 3 (10%) → 9.95 $A26Z/month
```

However, in `FeesManager.sol::calculateFeeAndNewAmountForOneTVS()`, the fee calculation incorrectly uses a cumulative `feeAmount` when subtracting per-flow fees:
```js
for (uint256 i; i < length; ++i) {
    feeAmount += calculateFeeAmount(feeRate, amounts[i]);
    newAmounts[i] = amounts[i] - feeAmount; // @audit - cumulative subtraction
}
```
Instead of subtracting each flow’s individual fee:
```js
newAmounts[i] = amounts[i] - calculateFeeAmount(feeRate, amounts[i]);
```
This causes the second flow to pay the fee for the first + second,
the third flow to pay the fee for 1 + 2 + 3,
and so on.

Tokens removed from flows do not match the fee transferred to the treasury, creating a mismatch between vesting accounting and actual token transfers.

### Internal Pre-conditions

1. A TVS exists with multiple vesting flows `allocation.amounts.length > 1`.
2. The user either splits or merges.
3. The contract is running the current incorrect implementation of `calculateFeeAndNewAmountForOneTVS()`.

### External Pre-conditions

None.

### Attack Path

Consider the following example: TVS (nftId => Allocation) and the allocation has 3 flows => 3 amounts [100e18, 200e18, 300e18] (i.e., [100, 200, 300] $A26Z) and a fee rate of 50 basis points (0.5%) as per whitepaper.

If the user attempts to split this TVS (e.g., into multiple NFTs), the protocol applies a 0.5% split fee per flow.

1. split fee rate = 0.5% = amount * 50 / 10,000
The correct fees should be:
```js
f0 = 0.5
f1 = 1
f2 = 1.5
```
2. However, the function computes cumulative fees:
```js
feeAmount after i=0 → 0.5
feeAmount after i=1 → 1.5 (0.5 + 1)
feeAmount after i=2 → 3 (1.5 + 1.5)
```
3. New amounts calculated:

```js
feeAmount += calculateFeeAmount(feeRate, amounts[i]);
newAmounts[i] = amounts[i] - feeAmount;

newAmounts[0] = 100 - 0.5 = 99.5
newAmounts[1] = 200 - 1.5 = 198.5 (should be 199e18)
newAmounts[2] = 300 - 3 = 297 (should be 298.5e18)
```

4. Total user tokens lost:
```js
Original total: 600
New total: 99.5 + 198.5 + 297 = 595
Total removed from user: 5, but should be only 3
```

5. The treasury receives only: `f0 + f1 + f2 = 3`

```js
token.safeTransfer(treasury, feeAmount);
```

### Impact

Users suffer silent permanent loss of vested tokens each time they use `splitTVS()` or `mergeTVS()`.

### PoC

Paste the test below inside `AlignerzVestingProtocolTest.sol` and run:
```bash
forge test --match-test test_IncorrectFeeMath_RemovesExtraTokens -vvvv
```

Also, you need to fix the other two mistakes in `calculateFeeAndNewAmountForOneTVS()`. Fixed:
```js
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) 
public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
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

Test:

```js
       function test_IncorrectFeeMath_RemovesExtraTokens() public {
        uint256[] memory amounts = new uint256[](3);
        // Use 18-decimals amounts (project token has 18 decimals)
        amounts[0] = 100 ether; // 100 * 1e18
        amounts[1] = 200 ether;
        amounts[2] = 300 ether;

        uint256 feeRate = 50; // 0.5% as per whitepaper (50 basis points)

        (uint256 feeAmount, uint256[] memory newAmounts) = vesting
            .calculateFeeAndNewAmountForOneTVS(feeRate, amounts, 3);

        // Log incorrect values
        emit log_named_uint("new[0]", newAmounts[0]); // should be 99.5 ether
        emit log_named_uint("new[1]", newAmounts[1]); // incorrect → 198.5 ether (should be 199 ether)
        emit log_named_uint("new[2]", newAmounts[2]); // incorrect → 297 ether (should be 298.5 ether)

        uint256 expectedTotalFee = (100 ether * 50) /
            10_000 +
            (200 ether * 50) /
            10_000 +
            (300 ether * 50) /
            10_000; // = 3 ether total fee expected

        uint256 sumOld = amounts[0] + amounts[1] + amounts[2]; // 600 ether
        uint256 sumNew = newAmounts[0] + newAmounts[1] + newAmounts[2];

        // Actual total removed = 5 ether, but should be only 3 ether
        assertEq(
            sumOld - sumNew,
            5 ether,
            "User lost 5 ether total, instead of 3"
        );

        // Treasury only receives the correct 3 ether
        assertEq(
            feeAmount,
            expectedTotalFee,
            "Treasury receives correct 3 ether"
        );
    }
```

Result:
```js
[PASS] test_IncorrectFeeMath_RemovesExtraTokens() (gas: 41809)
Logs:
  new[0]: 99500000000000000000
  new[1]: 198500000000000000000
  new[2]: 297000000000000000000

Traces:
  [41809] AlignerzVestingProtocolTest::test_IncorrectFeeMath_RemovesExtraTokens()
    ├─ [14184] ERC1967Proxy::fallback(50, [100000000000000000000 [1e20], 200000000000000000000 [2e20], 300000000000000000000 [3e20]], 3) [staticcall]
    │   ├─ [8905] AlignerzVesting::calculateFeeAndNewAmountForOneTVS(50, [100000000000000000000 [1e20], 200000000000000000000 [2e20], 300000000000000000000 [3e20]], 3) [delegatecall]
    │   │   └─ ← [Return] 3000000000000000000 [3e18], [99500000000000000000 [9.95e19], 198500000000000000000 [1.985e20], 297000000000000000000 [2.97e20]]
    │   └─ ← [Return] 3000000000000000000 [3e18], [99500000000000000000 [9.95e19], 198500000000000000000 [1.985e20], 297000000000000000000 [2.97e20]]
    ├─ emit log_named_uint(key: "new[0]", val: 99500000000000000000 [9.95e19])
    ├─ emit log_named_uint(key: "new[1]", val: 198500000000000000000 [1.985e20])
    ├─ emit log_named_uint(key: "new[2]", val: 297000000000000000000 [2.97e20])
    ├─ [0] VM::assertEq(5000000000000000000 [5e18], 5000000000000000000 [5e18], "User lost 5 ether total, instead of 3") [staticcall]
    │   └─ ← [Return]
    ├─ [0] VM::assertEq(3000000000000000000 [3e18], 3000000000000000000 [3e18], "Treasury receives correct 3 ether") [staticcall]
    │   └─ ← [Return]
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.96s (276.57µs CPU time)
```

### Mitigation

Replace cumulative fee subtraction with per-flow subtraction:
```js
newAmounts = new uint256[](length);

loop
uint256 perItemFee = calculateFeeAmount(feeRate, amounts[i]);
newAmounts[i] = amounts[i] - perItemFee;
feeAmount += perItemFee;
```
  