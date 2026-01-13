# [000438] Fee calculation is flawed
  
  ### Summary

Fee is calculated in `calculateFeeAndNewAmountForOneTVS` while merging or splitting the TVS. But due to flawed implementation of the function fee calculation is wrong

### Root Cause


```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
    }
}
```

From the above snippet, we can see that `feeAmount` is compounded during every iteration and then subtracted from the corresponding `amounts` element. The issue here is that the fee accumulates across iterations, even though it should not. As a result, the fee applied to the user becomes significantly higher than intended.

Consider the following example:

* fee = 1000 (10%)
* amounts = `[10,000, 10,000, 10,000]`
* length = 3

According to the current code, the resulting `newAmounts` would be:

```
[9000, 8000, 7000]
```

Here, the second and third amounts are effectively charged 20% and 30% fees, respectively, even though the intended fee is only 10%.


### Internal Pre-conditions

Users needs to call `splitTVS` or `mergeTVS`

### External Pre-conditions

None

### Attack Path

See Root Cause

### Impact

Users will be charged more than intended

### PoC

```solidity
    function calculateFeeAndNewAmountForOneTVS(
        uint256 feeRate,
        uint256[] memory amounts,
        uint256 length
    ) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint256[](length);
        for (uint256 i; i < length; i++) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

Replace the `FeesManger :: calculateFeeAndNewAmountForOneTVS` with the above code so that it will not be reverted.

Run 
```shell 
forge clean
forge test --match-test test_fee -vv
```
`POC :`


```solidity
    function test_fee() public {
        uint256 fee = 1000;
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 10000;
        amounts[1] = 10000;
        amounts[2] = 10000;
        uint256 length = amounts.length;
        (uint256 feeAmount, uint256[] memory newAmounts) = vesting
            .calculateFeeAndNewAmountForOneTVS(fee, amounts, length);
        console.log("New Amounts:");
        for (uint256 i = 0; i < newAmounts.length; i++) {
            console.log(newAmounts[i]);
        }
        console.log("Total Fee Amount:", feeAmount);
    }
```

`Logs :`

```shell
Ran 1 test for test/AlignerzVestingProtocolTest.t.sol:AlignerzVestingProtocolTest
[PASS] test_fee() (gas: 33050)
Logs:
  New Amounts:
  9000
  8000
  7000
  Total Fee Amount: 3000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.95s (2.74ms CPU time)

Ran 1 test suite in 5.96s (5.95s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)

```

### Mitigation

Fix the fee calculation so that fee is charged independently on each amount
  