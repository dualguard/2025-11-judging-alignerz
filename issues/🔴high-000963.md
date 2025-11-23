# [000963] Users will not receive their dividends due to OOG
  
  ### Summary

The `getUnclaimedAmounts` function uses continue without incrementing the i counter. If `claimedFlows[i] == true` or `claimedSeconds[i] == 0`, the counter is not incremented, resulting in an infinite loop and DoS. This blocks the distribution of dividends. 

### Root Cause

In a for loop, the counter i is incremented only at the end of the block, but continue is executed earlier, skipping the increment:
```solidity 
for (uint i; i < len; ) {

>>> if (claimedFlows[i]) continue; 
    if (claimedSeconds[i] == 0) {     
       amount += amounts[i];
>>>    continue; 
    }
//...code 
    unchecked {
        ++i; 
    }
}
```

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. The user holds TVS tokens and wants to receive dividends.
2. Calls this function to distribute undistributed tokens in order to receive their share of dividends.

### Impact

Users will not receive their dividends 

### PoC

```solidity 
 function test_OOG() public {
        for (uint i; i < 10; ) {
            if (i == 0) continue;
            unchecked {
                ++i;
            }
        }
    }
```

```bash
forge test --mt test_OOG -vv

Ran 1 test for test/AlignerzVestingProtocolTest.t.sol:AlignerzVestingProtocolTest
[FAIL: EvmError: OutOfGas] test_OOG() (gas: 1073720760)
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 6.14s (2.94s CPU time)

Ran 1 test suite in 6.15s (6.14s CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/AlignerzVestingProtocolTest.t.sol:AlignerzVestingProtocolTest
[FAIL: EvmError: OutOfGas] test_OOG() (gas: 1073720760)

Encountered a total of 1 failing tests, 0 tests succeeded
```

### Mitigation

_No response_
  