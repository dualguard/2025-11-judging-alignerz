# [000187] Missing array initialization in `_computeSplitArrays` function leads to systematic array out-of-bounds panic.
  
  ### Summary

The `_computeSplitArrays` function is used in `splitTVS` function to transform an allocation into another allocation where all flows in it contain a certain percentage of the amounts of the original TVS. It is defined as follows:

```solidity
    function _computeSplitArrays(Allocation storage allocation, uint256 percentage, uint256 nbOfFlows)
        internal
        view
        returns (Allocation memory alloc)
    {
        uint256[] memory baseAmounts = allocation.amounts;
        uint256[] memory baseVestings = allocation.vestingPeriods;
        uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
        uint256[] memory baseClaimed = allocation.claimedSeconds;
        bool[] memory baseClaimedFlows = allocation.claimedFlows;
        alloc.assignedPoolId = allocation.assignedPoolId;
        alloc.token = allocation.token;
        for (uint256 j; j < nbOfFlows;) {
            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
            alloc.vestingPeriods[j] = baseVestings[j];
            alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
            alloc.claimedSeconds[j] = baseClaimed[j];
            alloc.claimedFlows[j] = baseClaimedFlows[j];
            unchecked {
                ++j;
            }
        }
    }
```

The issue is that the return value `Allocation memory alloc` is a struct that contains memory arrays and they are never initialized. This means an array out-of-bounds will occur at the line:

```solidity
alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
```

### Root Cause

The root cause is uninitialized memory arrays that leads to systematic array out-of-bounds panic.

### Attack Path

There is no specific attack path, simply this bug results in a complete DOS of the `splitTVS` function.

### Impact

The impact of this issue is high as it breaks entirely one of the main features of the protocol.

### PoC

Please copy paste the following test in the test file:

```solidity
    function test_SplitFailDueToUninitializedArray() public {
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
    }
```

This test shows that the split feature don't work and revert.
Note that other issues in the codebase (uninitialized memory arrays in `calculateFeeAndNewAmountForOneTVS` but also infinite loop) will trigger a failure. But if we correct other issues, the logs of the test show the issue:

```solidity
    │   ├─ [37882] AlignerzVesting::splitTVS(0, [5000, 5000], 1) [delegatecall]
    │   │   ├─ [4318] AlignerzNFT::extOwnerOf(1) [staticcall]
    │   │   │   └─ ← [Return] bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083]
    │   │   ├─ [9944] Aligners26::transfer(ECRecover: [0x0000000000000000000000000000000000000001], 0)
    │   │   │   ├─ emit Transfer(from: ERC1967Proxy: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], to: ECRecover: [0x0000000000000000000000000000000000000001], value: 0)
    │   │   │   └─ ← [Return] true
    │   │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    └─ ← [Revert] panic: array out-of-bounds access (0x32)
```

### Mitigation

Make sure to correctly initialize all the memory arrays inside the memory `Allocation`:

```solidity
    function _computeSplitArrays(Allocation storage allocation, uint256 percentage, uint256 nbOfFlows)
        internal
        view
        returns (Allocation memory alloc)
    {
        // Initialize arrays with the correct length!
        alloc.amounts = new uint256[](nbOfFlows);
        alloc.vestingPeriods = new uint256[](nbOfFlows);
        alloc.vestingStartTimes = new uint256[](nbOfFlows);
        alloc.claimedSeconds = new uint256[](nbOfFlows);
        alloc.claimedFlows = new bool[](nbOfFlows);

        alloc.assignedPoolId = allocation.assignedPoolId;
        alloc.token = allocation.token;

        for (uint256 j; j < nbOfFlows;) {
            alloc.amounts[j] = (allocation.amounts[j] * percentage) / BASIS_POINT;
            alloc.vestingPeriods[j] = allocation.vestingPeriods[j];
            alloc.vestingStartTimes[j] = allocation.vestingStartTimes[j];
            alloc.claimedSeconds[j] = allocation.claimedSeconds[j];
            alloc.claimedFlows[j] = allocation.claimedFlows[j];
            unchecked {
                ++j;
            }
        }
    }
```
  