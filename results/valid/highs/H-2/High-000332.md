# [000332] Uninitialized Array in `calculateFeeAndNewAmountForOneTVS`
  
  ### Summary


The [ calculateFeeAndNewAmountForOneTVS](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169) function in `FeesManager.sol` contains a critical bug where the return parameter `newAmounts` array is never initialized before being accessed. This causes the function to always revert with a `Panic(0x32)` (array out-of-bounds access) error, making the entire `mergeTVS` functionality completely unusable. Any user attempting to merge TVS NFTs will experience transaction failures, preventing them from consolidating their vesting positions.


### Root Cause


The root cause is improper initialization of the `newAmounts` return parameter in the `calculateFeeAndNewAmountForOneTVS` function.
```javascript

  function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    ...
     --->       newAmounts[i] = amounts[i] - feeAmount;
    ...
    }
```

In Solidity, return parameters are automatically initialized to their default values:
- `uint256 feeAmount` → initialized to `0` 
- `uint256[] memory newAmounts` → initialized to `[]` (empty array with length 0) 

The function attempts to write to `newAmounts[i]` without first allocating memory for the array elements. When Solidity performs bounds checking (`i < newAmounts.length`), it finds `0 < 0` evaluates to false, triggering a panic revert.


### Internal Pre-conditions

.

### External Pre-conditions

.

### Attack Path

 This is not an attack but a critical functionality bug. The normal user flow that triggers the bug:

1. User (Alice) owns multiple TVS NFTs:
   - NFT #100: 1 flow with 10,000 tokens
   - NFT #200: 1 flow with 5,000 tokens

2. Alice attempts to merge NFT #200 into NFT #100 by calling:
   ```javascript
   mergeTVS(projectId: 1, mergedNftId: 100, projectIds: [2], nftIds: [200])
   ```

3. Inside `mergeTVS()` calls:
   ```javascript
   (uint256 feeAmount, uint256[] memory newAmounts) = 
       calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
   ```

4. Inside `calculateFeeAndNewAmountForOneTVS()`:
   - Loop iteration 0: `i = 0`
   - `feeAmount` calculation succeeds 
   -  Attempts `newAmounts[0] = ...`
   - Bounds check: `0 < 0` (index < array length) → fails
   - **Transaction reverts with `Panic(0x32)`** 

5. Alice's merge transaction fails completely


### Impact


- The entire `mergeTVS` and `splitTVS` functionality is **completely broken** and unusable
- Users cannot consolidate their TVS NFTs
- Users are forced to manage multiple separate NFTs instead of merging them
- **100% failure rate** for any merge attempt



### PoC

Place below test at `AlignerzVestingProtocolTest` test file.

**Run the test:**
```javascript
forge test --match-test test_UninitializedArrayInCalculateFee -vvvv
```
Here is the test:
```javascript
   function test_UninitializedArrayInCalculateFee() public {
        // Setup: Creating a simple amounts array with 1 element
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 10_000 ether;
        
        uint256 feeRate = 200; // 2% in basis points
        uint256 length = 1;

        // Expected: This should revert because newAmounts is an empty array (length 0)
        // and accessing newAmounts[0] causes an array out of bounds panic (Panic(0x32))
        vm.expectRevert(stdError.indexOOBError);
        vesting.calculateFeeAndNewAmountForOneTVS(feeRate, amounts, length);
    }

```    


**Test Output:**
```javascript
1 test for test/AlignerzVestingProtocolTest.t.sol:AlignerzVestingProtocolTest
[PASS] test_UninitializedArrayInCalculateFee() (gas: 23885)
Traces:
  [23885] AlignerzVestingProtocolTest::test_UninitializedArrayInCalculateFee()
    ├─ [0] VM::expectRevert(custom error 0xf28dceb3:  $NH{q2)
    │   └─ ← [Return]
    ├─ [9170] ERC1967Proxy::fallback(200, [10000000000000000000000 [1e22]], 1) [staticcall]
    │   ├─ [3914] AlignerzVesting::calculateFeeAndNewAmountForOneTVS(200, [10000000000000000000000 [1e22]], 1) [delegatecall]
    │   │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.31s (269.44µs CPU time)
```

### Mitigation

 Initialize the `newAmounts` array with the correct size before accessing it.
  