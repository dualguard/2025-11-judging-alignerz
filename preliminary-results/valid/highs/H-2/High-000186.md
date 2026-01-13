# [000186] `newAmounts` memory array is not initialized in `calculateFeeAndNewAmountForOneTVS` function, leading to systematic array out-of-bounds panic.
  
  ### Summary

The calculateFeeAndNewAmountForOneTVS function, defined in FeesManager contract, is called by both mergeTVS and splitTVS functions and is defined as follows:

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

Multiple findings are related to this function (missing loop incrementation, `feeAmount` subtraction which is incorrect), but the problem with the current finding is that the `newAmounts` memory array is never initialized. This means that the line `newAmounts[i] = amounts[i] - feeAmount;` will fail.

### Root Cause

Uninitialized memory array leading to systematic memory out-of-bounds panic.

### Attack Path

There is no specific attack path. Any attempt to merge or split a TVS will result in a failure due to this array out-of-bounds panic.

### Impact

The impact of this issue is high as it breaks entirely some of the main features of the protocol.

### PoC

Please copy paste the following test in the test file:

```solidity
    function test_SplitAndMerge_FailDueToUninitializedArray() public {
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

        // 7. Test splitTVS - will fail due to out of bounds panic
        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5000; // 50%
        percentages[1] = 5000; // 50%

        vm.prank(bidders[0]);
        vm.expectRevert(); // out of bounds panic
        vesting.splitTVS(PROJECT_ID, percentages, nftId1);

        // 8. Transfer nftId2 from bidders[1] to bidders[0] for merge test
        vm.prank(bidders[1]);
        nft.transferFrom(bidders[1], bidders[0], nftId2);

        // 9. Test mergeTVS - will also fail due to out of bounds panic
        uint256[] memory projectIds = new uint256[](1);
        projectIds[0] = PROJECT_ID;

        uint256[] memory nftIds = new uint256[](1);
        nftIds[0] = nftId2;

        vm.prank(bidders[0]);
        vm.expectRevert(); // out of bounds panic
        vesting.mergeTVS(PROJECT_ID, nftId1, projectIds, nftIds);
    }
```

When looking at the log, we can see the actual error, which is an `array out-of-bounds access` for both `mergeTVS` and `splitTVS` functions:

```solidity
 ├─ [14961] ERC1967Proxy::fallback(0, [5000, 5000], 1)
    │   ├─ [14199] AlignerzVesting::splitTVS(0, [5000, 5000], 1) [delegatecall]
    │   │   ├─ [4318] AlignerzNFT::extOwnerOf(1) [staticcall]
    │   │   │   └─ ← [Return] bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083]
    │   │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    ├─ [0] VM::prank(bidder1: [0xb549357D1EA5b44c8F1d82046c7dfD6ed2265A8B])
    │   └─ ← [Return]
    ├─ [20122] AlignerzNFT::transferFrom(bidder1: [0xb549357D1EA5b44c8F1d82046c7dfD6ed2265A8B], bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083], 2)
    │   ├─ emit Approval(owner: bidder1: [0xb549357D1EA5b44c8F1d82046c7dfD6ed2265A8B], approved: 0x0000000000000000000000000000000000000000, tokenId: 2)
    │   ├─ emit Transfer(from: bidder1: [0xb549357D1EA5b44c8F1d82046c7dfD6ed2265A8B], to: bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083], tokenId: 2)
    │   └─ ← [Return]
    ├─ [0] VM::prank(bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083])
    │   └─ ← [Return]
    ├─ [0] VM::expectRevert(custom error 0xf4844814)
    │   └─ ← [Return]
    ├─ [14697] ERC1967Proxy::fallback(0, 1, [0], [2])
    │   ├─ [13923] AlignerzVesting::mergeTVS(0, 1, [0], [2]) [delegatecall]
    │   │   ├─ [4318] AlignerzNFT::extOwnerOf(1) [staticcall]
    │   │   │   └─ ← [Return] bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083]
    │   │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
```

### Mitigation

Ensure that `calculateFeeAndNewAmountForOneTVS` is correctly implemented by initializing the memory array (and correcting other issues of course):

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
        public
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
        newAmounts = new uint256[](length); // @audit initialize memory array

        for (uint256 i; i < length;) {

           ...

            // @audit increment counter (other finding)
            unchecked {
                ++i;
            }
        }
    }
  