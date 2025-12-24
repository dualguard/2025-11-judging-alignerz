# [000331] Cumulative Fee Calculation Error in `calculateFeeAndNewAmountForOneTVS`
  
  ### Summary

The [calculateFeeAndNewAmountForOneTVS](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169) function in `FeesManager.sol` contains a logic error where it subtracts the cumulative total fee from each flow instead of subtracting only the individual flow's fee. This results in exponentially increasing overcharges for flows with higher indices. When merging TVS or Spliting TVS NFTs with multiple flows, users lose significantly more tokens than the intended fee percentage, with the overcharge increasing for each subsequent flow. For example, with a 2% fee on 5 flows, users can lose up to 6% of their tokens instead of the intended 2%, resulting in a 3x overcharge.


### Root Cause


The root cause is an incorrect fee deduction logic on line 172 of `FeesManager.sol` in the `calculateFeeAndNewAmountForOneTVS` function.

```javascript

  function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    ...
                feeAmount += calculateFeeAmount(feeRate, amounts[i]);  
     --->       newAmounts[i] = amounts[i] - feeAmount;
    ...
    }
```


**The Issue:**
- calculates the individual fee for the current flow and adds it to the cumulative `feeAmount` variable
- incorrectly subtracts the **cumulative** `feeAmount` (sum of all fees so far) from the current flow
- Each flow should only have its own individual fee deducted, not the accumulated fees from all previous flows



### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path
Same scenario can happen with splitTVS also.
**User Setup:**
   - Alice owns NFT #100 with 3 flows:
     - Flow 0: 10,000 tokens
     - Flow 1: 5,000 tokens  
     - Flow 2: 3,000 tokens
   - Alice owns NFT #200 with 1 flow: 2,000 tokens
   - Merge fee rate: 2% (200 basis points)

2. **Alice initiates merge:**
   ```solidity
   mergeTVS(projectId: 1, mergedNftId: 100, projectIds: [2], nftIds: [200])
   ```

3. **Inside `mergeTVS()`:**
   ```solidity
   (uint256 feeAmount, uint256[] memory newAmounts) = 
       calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
   ```
   - Called with NFT #100's 3 flows
   - `amounts = [10,000, 5,000, 3,000]`
   - `length = 3`

4. **Inside `calculateFeeAndNewAmountForOneTVS()`:**
   
   **Iteration 0:**
   - Calculate fee: `200` (2% of 10,000)
   - `feeAmount = 200` (cumulative)
   - `newAmounts[0] = 10,000 - 200 = 9,800` ✓
   
   **Iteration 1:**
   - Calculate fee: `100` (2% of 5,000)
   - `feeAmount = 300` (cumulative: 200 + 100)
   - `newAmounts[1] = 5,000 - 300 = 4,700` (should be 4,900)
   - **Alice loses extra 200 tokens**
   
   **Iteration 2:**
   - Calculate fee: `60` (2% of 3,000)
   - `feeAmount = 360` (cumulative: 200 + 100 + 60)
   - `newAmounts[2] = 3,000 - 360 = 2,640` (should be 2,940)
   - **Alice loses extra 300 tokens**

5. **updates storage:**
   ```solidity
   mergedTVS.amounts = newAmounts;  // [9,800, 4,700, 2,640]
   ```

6. **Result:**
   - Alice's NFT #100 now has: `[9,800, 4,700, 2,640]` = 17,140 tokens
   - Should have: `[9,800, 4,900, 2,940]` = 17,640 tokens
   - **Alice permanently loses 500 tokens** (2.78% instead of 2%)


### Impact

- Incorrect fee collection mechanism that overcharges users
- **Permanent loss of user tokens** beyond the stated fee percentage
- Loss of user trust when they discover they're losing more tokens than advertised
- The more flows an NFT has, the worse the overcharge becomes
- Creates unfair advantage for users with single-flow NFTs vs multi-flow NFTs


### PoC

_No response_

### Mitigation

 Calculate and subtract only the individual fee for each flow, not the cumulative total.
  