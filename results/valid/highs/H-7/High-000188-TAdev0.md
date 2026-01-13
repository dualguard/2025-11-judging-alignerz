# [000188] Infinite loop in `calculateFeeAndNewAmountForOneTVS` function breaks TVS merge and split features.
  
  ### Summary

The  `calculateFeeAndNewAmountForOneTVS` function, defined in _FeesManager_ contract, is called by both `mergeTVS` and `splitTVS` functions and is defined as follows:

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
        public
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

Multiple findings are related to this function (`newAmounts` memory array not initialized leading to out of bounds panic, `feeAmount` subtraction which is incorrect), but the problem with the current finding is that the loop counter `i` is never incremented.

This will lead to an infinite loop, ultimately triggering an out of gas panic.

### Root Cause

The root cause is the absence of counter increment in the `for` loop in `calculateFeeAndNewAmountForOneTVS` function.


### Impact

The impact of this issue is high as it breaks entirely some of the main features of the protocol.

### PoC

Please copy paste the following test in _AlignerzVestingProtocolTest_ file:

```solidity
 function test_SplitAndMerge_FailDueToInfiniteLoop() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // 1. Launch project
        vesting.launchBiddingProject(
            address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", false
        );

        // 2. Create pool
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);

        vm.stopPrank();

        // 3. Place bids
        vm.startPrank(bidders[0]);
        usdt.approve(address(vesting), BIDDER_USD * 2); // Approve for 2 bids
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        vm.startPrank(bidders[1]);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        // 4. Generate merkle proofs for both bids
        bytes32[] memory leaves = new bytes32[](2);
        leaves[0] = getLeaf(bidders[0], BIDDER_USD, PROJECT_ID, 0);
        leaves[1] = getLeaf(bidders[1], BIDDER_USD, PROJECT_ID, 0);

        CompleteMerkle m = new CompleteMerkle();
        bytes32 root = m.getRoot(leaves);

        bytes32[] memory proof0 = m.getProof(leaves, 0);
        bytes32[] memory proof1 = m.getProof(leaves, 1);

        // 5. Finalize bids
        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = root;

        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);

        // 6. Claim NFTs
        vm.prank(bidders[0]);
        uint256 nftId1 = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, proof0);

        vm.prank(bidders[1]);
        uint256 nftId2 = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, proof1);

        // 7. Test splitTVS - will fail due to infinite loop
        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5000; // 50%
        percentages[1] = 5000; // 50%

        vm.prank(bidders[0]);
        vm.expectRevert(); // Out of gas due to infinite loop
        vesting.splitTVS(PROJECT_ID, percentages, nftId1);

        // 8. Transfer nftId2 from bidders[1] to bidders[0] for merge test
        vm.prank(bidders[1]);
        nft.transferFrom(bidders[1], bidders[0], nftId2);

        // 9. Test mergeTVS - will also fail due to infinite loop
        uint256[] memory projectIds = new uint256[](1);
        projectIds[0] = PROJECT_ID;

        uint256[] memory nftIds = new uint256[](1);
        nftIds[0] = nftId2;

        vm.prank(bidders[0]);
        vm.expectRevert(); // Out of gas due to infinite loop
        vesting.mergeTVS(PROJECT_ID, nftId1, projectIds, nftIds);
    }
```

This test shows that both the split feature and the merge feature don't work and revert.
Note that other issues in the codebase (uninitialized memory arrays in `calculateFeeAndNewAmountForOneTVS` but also in `_computeSplitArrays`) need to be corrected so that an out of gas error is actually triggered.
But if both arrays are correctly initialized, an out of gas error is triggered:

```solidity
├─ [1037862435] ERC1967Proxy::fallback(0, [5000, 5000], 1)
    │   ├─ [1037861678] AlignerzVesting::splitTVS(0, [5000, 5000], 1) [delegatecall]
    │   │   ├─ [4318] AlignerzNFT::extOwnerOf(1) [staticcall]
    │   │   │   └─ ← [Return] bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083]
    │   │   └─ ← [OutOfGas] EvmError: OutOfGas
    │   └─ ← [Revert] EvmError: Revert
    └─ ← [Revert] EvmError: Revert
```

### Mitigation

Ensure that `calculateFeeAndNewAmountForOneTVS` is correctly implemented with `i` counter incremented during the `for` loop:

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
        public
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
        newAmounts = new uint256[](length); // @audit initialize memory array (other finding)

        for (uint256 i; i < length;) {

           ...

            // @audit increment counter
            unchecked {
                ++i;
            }
        }
    }
  