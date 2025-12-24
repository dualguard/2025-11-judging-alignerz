# [000673] [H-7] Incorrect Split Calculation in splitTVS leading to Fund Loss
  
  ### Summary

The `splitTVS` function incorrectly updates the original NFT's allocation *before* calculating the allocation for the new NFT. This results in the new NFT receiving a percentage of the *reduced* amount, leading to a permanent loss of funds.

### Root Cause

In `protocol/src/contracts/vesting/AlignerzVesting.sol:1082-1104`, the loop iterates through the percentages. For the first iteration (`i=0`), it updates the original NFT (`allocationOf[splitNftId]`) with the new reduced amount. For subsequent iterations (`i>0`), it calls `_computeSplitArrays(allocation, ...)` passing the *already reduced* `allocation` storage pointer. `_computeSplitArrays` then calculates the new amount based on this reduced value.

### Internal Pre-conditions

  1. User owns a vesting NFT.
  2. User calls `splitTVS` with at least two percentages (e.g., 50%, 50%).


### External Pre-conditions

None. 

### Attack Path

  1. User has an NFT with 100 tokens.
  2. User calls `splitTVS` with `[5000, 5000]` (50%, 50%).
  3. Iteration 0: Original NFT amount is updated to 50% of 100 = 50.
  4. Iteration 1: New NFT amount is calculated as 50% of the *current* storage amount (50) = 25.
  5. Total allocated: 50 + 25 = 75. 25 tokens are lost.

### Impact

Permanent loss of funds for users splitting their NFTs. In a 50/50 split, 25% of the total value is lost.


### PoC

  ```solidity
  // protocol/test/SplitTVSFundLossTest.t.sol
  function test_splitTVS_fundLoss() public {
      uint256[] memory percentages = new uint256[](2);
      percentages[0] = 5000; // 50%
      percentages[1] = 5000; // 50%
      
      // Assume original NFT (ID 1) has 100 ether
      uint256 originalNftId = 1; 
      
      vm.prank(bidder);
      (uint256 splitNftId, uint256[] memory newNftIds) = vesting.splitTVS(projectId, percentages, originalNftId);
      
      // Fast forward to allow claiming
      vm.warp(block.timestamp + 2 days); 
      
      // Claim tokens for original NFT
      vm.prank(bidder);
      vesting.claimTokens(projectId, splitNftId);
      uint256 amount0 = token.balanceOf(bidder);
      
      // Claim tokens for new NFT
      vm.prank(bidder);
      vesting.claimTokens(projectId, newNftIds[0]);
      uint256 totalClaimed = token.balanceOf(bidder);
      uint256 amount1 = totalClaimed - amount0;
      
      // Expected: 50 + 50 = 100
      // Actual: 50 + 25 = 75
      assertEq(amount0, 50 ether);
      assertEq(amount1, 25 ether); 
      assertEq(totalClaimed, 75 ether);
  }
  ```


### Mitigation

  1. **Fix Issue #3 first** (array allocation / loop increment) so `splitTVS` can execute. Reference patch below:
     ```diff
     // protocol/src/contracts/vesting/feesManager/FeesManager.sol
     -for (uint256 i; i < length;) {
     -    feeAmount += calculateFeeAmount(feeRate, amounts[i]);
     -    newAmounts[i] = amounts[i] - feeAmount;
     -}
     +newAmounts = new uint256[](length);
     +for (uint256 i; i < length; i++) {
     +    uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
     +    feeAmount += fee;
     +    newAmounts[i] = amounts[i] - fee;
     +}
     ```
     ```diff
     // protocol/src/contracts/vesting/AlignerzVesting.sol (_computeSplitArrays)
     +alloc.amounts = new uint256[](nbOfFlows);
     +alloc.vestingPeriods = new uint256[](nbOfFlows);
     +alloc.vestingStartTimes = new uint256[](nbOfFlows);
     +alloc.claimedSeconds = new uint256[](nbOfFlows);
     +alloc.claimedFlows = new bool[](nbOfFlows);
     ```
  2. Read the original allocation into memory *before* the loop and use the memory values for calculations. Do not update the storage of the original NFT until *after* all calculations are done, or pass the initial memory struct to `_computeSplitArrays`.

  