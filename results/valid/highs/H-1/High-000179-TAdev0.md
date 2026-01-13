# [000179] `allocation` of the TVS to split is not loaded to memory in `splitTVS` function, breaking the split mechanism, leading to loss of user rewards.
  
  ### Summary

`splitTVS` function allows a user to split a given TVS into any number of TVS. The `uint256[] calldata percentages` argument allows to choose the ponderation of each NFT.

The issue arises in the loop  when `_computeSplitArrays` is called.  During the first iteration, `_assignAllocation` function modifies the original allocation for the NFT because `newAlloc` in loop and `allocation` outside the loop point to the same storage variable. This means:
- the original allocation data is modified during the first iteration and reduced to what percentage it should be
- subsequent iterations use the corrupted data in `_computeSplitArrays(allocation, percentage, nbOfFlows)` instead of using a memory copy

The split calculations become incorrect for iterations `i >= 1`.

### Root Cause

The `splitTVS` function is wrongly implemented:

```solidity
     (Allocation storage allocation, IERC20 token) = isBiddingProject
            ? (biddingProjects[projectId].allocations[splitNftId], biddingProjects[projectId].token)
            : (rewardProjects[projectId].allocations[splitNftId], rewardProjects[projectId].token);

    ...

   for (uint256 i; i < nbOfTVS;) {
            uint256 percentage = percentages[i];
            sumOfPercentages += percentage;

            uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);
            if (i != 0) newNftIds[i - 1] = nftId;
            Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
            NFTBelongsToBiddingProject[nftId] = isBiddingProject ? true : false;
            Allocation storage newAlloc = isBiddingProject
                ? biddingProjects[projectId].allocations[nftId]
                : rewardProjects[projectId].allocations[nftId];
            _assignAllocation(newAlloc, alloc); 
            allocationOf[nftId] = newAlloc;
     ...
```

Before the loop, `allocation` storage variable should be loaded to memory. That way, we can use it in the loop here:

```solidity
            // @audit use a memory copy of `allocation` in `_computeSplitArrays`
            Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
```

### Attack Path

There is no specific attack path. Any split in more than 2 NFTs will result in loss of rewards for users.

### Impact

The impact of this issue is high as it results in loss of rewards for users.

### PoC

Please copy paste the following test in the test file:

```solidity
 function testSplitTVSLossOfFlowAmounts() external {
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

        // 5. Finalize bids
        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = root;

        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);

        // 6. Claim NFTs
        vm.prank(bidders[0]);
        uint256 nftId1 = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, proof0);
        console.log("NFT ID 1:", nftId1);

        // 7. Test splitTVS
        uint256[] memory percentages = new uint256[](4);
        percentages[0] = 2500; // 25%
        percentages[1] = 2500;
        percentages[2] = 2500;
        percentages[3] = 2500;

        vm.prank(bidders[0]);
        vesting.splitTVS(PROJECT_ID, percentages, nftId1);

        // 8. Show that flow amounts are incorrect
        for (uint256 i = 0; i < 4; i++) {
            uint256 splitNftId = i + 1;
            uint256[] memory amounts = vesting.getAllocationAmountsForTest(splitNftId);
            uint256 amount = amounts[0];
            console.log("NFT ID %d, Flow %d, Amount: %d", splitNftId, i, amount);
        }
    }
```

In order to be able to run this test, you will need to add one getter function in the AlignerzVesting contract:

```solidity
    // FOR TEST
    function getAllocationAmountsForTest(uint256 nftId) external view returns (uint256[] memory) {
        return allocationOf[nftId].amounts;
    }
```

You will also need to rectify issues in `calculateFeeAndNewAmountForOneTVS` function (missing increment, memory array uninitialized) and in `_computeSplitArrays` function (memory arrays uninitialized):

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
        public
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
        newAmounts = new uint256[](length); // FOR TEST

        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;

            // FOR TEST
            unchecked {
                ++i;
            }
        }
    }
```

```solidity
 // FOR TEST
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

The output of the test is:

```solidity
  NFT ID 1, Flow 0, Amount: 250000000000000000000
  NFT ID 2, Flow 1, Amount: 62500000000000000000
  NFT ID 3, Flow 2, Amount: 62500000000000000000
  NFT ID 4, Flow 3, Amount: 62500000000000000000
```

What actually happens is:
- a user placed a bid of 1000 * 10**18 
- `finalizeBid` is called
- user claims his nft
- user splits his NFT into 4 equivalents NFTS (25% ponderation each)
- but the result is that the first NFT has indeed 1/4 of the value, but the 3 others have 1/4 of 1/4 == 1/16 of the value

This means the user lost more than half of the value of his NFT.

### Mitigation

Revisit the implementation of `splitTVS` so that it correctly divides the `amounts` array when splitting a TVS.
For that, loading the storage allocation into memory and use this copy to compute the amount of the newly minted TVS.
  