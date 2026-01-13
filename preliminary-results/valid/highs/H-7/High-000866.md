# [000866] Denial of Service (DoS) on splitTVS and mergeTVS due to unupdated loop index
  
  ### Summary

Both functions `mergeTVS()` and `splitTVS()` call [calculateFeeAndNewAmountForOneTVS()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174). In this function, the loop index i is not updated inside the for loop. As a result, the loop runs indefinitely until the transaction runs out of gas, causing the calling functions to revert.

### Root Cause

In `FeesManager::calculateFeeAndNewAmountForOneTVS()`, the for loop uses the index variable i but never increments it. This leads to an infinite loop and eventually an out-of-gas error when executing `splitTVS()` or `mergeTVS()`.

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

None

### Impact

- Users cannot split or merge TVS.
- Core vesting functionalities are blocked.

### PoC

Consider adding the following test to the `AlignerzVestingProtocolTest.t.sol` and call it via:
`forge test --mt testRevertOnSplitTVS -vvv`

```solidity
function testRevertOnSplitTVS() external {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // 1. Launch project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);

        // 2. Create multiple pools with different prices
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.02 ether, false);
        vesting.createPool(PROJECT_ID, 4_000_000 ether, 0.03 ether, false);

        vesting.addUsersToWhitelist(bidders, PROJECT_ID);
        vm.stopPrank();
        for (uint256 i = 0; i < NUM_BIDDERS; i++) {
            vm.startPrank(bidders[i]);

            // Approve and place bid
            usdt.approve(address(vesting), BIDDER_USD);

            // Different vesting periods to test variety
            uint256 vestingPeriod = (i % 3 == 0) ? 90 days : (i % 3 == 1) ? 180 days : 365 days;

            vesting.placeBid(PROJECT_ID, BIDDER_USD, vestingPeriod);
            vm.stopPrank();
        }

         // 5. Prepare bid allocations (this would be done off-chain)
        BidInfo[] memory allBids = new BidInfo[](NUM_BIDDERS);

        // Simulate off-chain allocation process
        for (uint256 i = 0; i < NUM_BIDDERS; i++) {
            // Assign bids to pools (in reality, this would be based on some algorithm)
            uint256 poolId = i % 3;
            bool accepted = i < 15; // First 15 bidders are accepted

            allBids[i] = BidInfo({
                bidder: bidders[i],
                amount: BIDDER_USD, // For simplicity, all bids are the same amount
                vestingPeriod: (i % 3 == 0) ? 90 days : (i % 3 == 1) ? 180 days : 365 days,
                poolId: poolId,
                accepted: accepted
            });
        }

        // 6. Generate merkle roots for each pool
        bytes32[] memory poolRoots = new bytes32[](3);
        for (uint256 poolId = 0; poolId < 3; poolId++) {
            poolRoots[poolId] = generateMerkleProofs(allBids, poolId);
        }

        // 7. Generate refund proofs
        refundRoot = generateRefundProofs(allBids);

        // 8. Finalize project with merkle roots
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, refundRoot, poolRoots, 60);
        uint256[] memory nftIds = new uint256[](15);
        // uint256[] memory newNfts;
        address bidder = bidders[0];
        uint256 poolId = bidderPoolIds[bidder];
        vm.prank(bidder);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, poolId, BIDDER_USD, bidderProofs[bidder]);
        nftIds[0] = nftId;
        uint256[] memory percentages = new uint256[](5);
        for (uint8 i = 0; i < 5; i++)
        {
            percentages[i] = 2000;
        }
        vm.prank(bidder);
        (uint256 oldNft, uint256[] memory newNfts) = vesting.splitTVS(PROJECT_ID, percentages, nftId);
    }
```

Which will give the follwing output:
```bash
AlignerzVesting::splitTVS(0, [2000, 2000, 2000, 2000, 2000], 1) [delegatecall]
    │   │   ├─ [4318] AlignerzNFT::extOwnerOf(1) [staticcall]
    │   │   │   └─ ← [Return] bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083]
    │   │   └─ ← [OutOfGas] EvmError: OutOfGas
    │   └─ ← [Revert] EvmError: Revert
    └─ ← [Revert] EvmError: Revert

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 3.93s (2.47s CPU time)

Ran 1 test suite in 3.93s (3.93s CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/AlignerzVestingProtocolTest.t.sol:AlignerzVestingProtocolTest
[FAIL: EvmError: Revert] testRevertOnSplitTVS()
```

Note: Ensure memory allocation issues in splitTVS are resolved prior to testing the loop index problem.

### Mitigation

Update the loop index in `calculateFeeAndNewAmountForOneTVS()`:

```diff
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
+          unchecked {
+                ++i;
+           }
        }
}

``` 
  