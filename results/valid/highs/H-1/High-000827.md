# [000827] Base allocation mutated before split computation, causing all TVS splits to become inaccurate
  
  ### Summary

Each split calculation modifies the base allocation, causing subsequent splits to use already-reduced amounts instead of the original amounts.

### Root Cause


When the first iteration (i=0) assigns the split allocation to `splitNftId`, it **modifies the base allocation**. Then next iterations (i > 0) read the **already-reduced amounts** instead of the original amounts.

[AlignerzVesting.sol:1054-1107](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054C1-L1107C6)

[AlignerzVesting.sol:1113-1143](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113-L1143)

```solidity
function splitTVS(...) external returns (uint256, uint256[] memory) {
    //...
    (Allocation storage allocation, IERC20 token) = isBiddingProject ?
        (biddingProjects[projectId].allocations[splitNftId], biddingProjects[projectId].token) :
        (rewardProjects[projectId].allocations[splitNftId], rewardProjects[projectId].token);
    //...
    for (uint256 i; i < nbOfTVS;) {
        //...
        uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);
        //...
        Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
        //...
        Allocation storage newAlloc = isBiddingProject ? biddingProjects[projectId].allocations[nftId] : rewardProjects[projectId].allocations[nftId];
        _assignAllocation(newAlloc, alloc);
        allocationOf[nftId] = newAlloc;
        allAlloc[i].amounts = alloc.amounts;
        //...
    }
    //...
}
```

```solidity
function _computeSplitArrays(
    Allocation storage allocation,
    uint256 percentage,
    uint256 nbOfFlows
) internal view returns (Allocation memory alloc) {
    uint256[] memory baseAmounts = allocation.amounts;
    for (uint256 j; j < nbOfFlows;) {
        alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
        // ...
    }
}
```


### Internal Pre-conditions

This bug occurs in every split operation with `percentages.length > 1` (i.e. every practical splits)

### External Pre-conditions

None

### Attack Path

1. User owns an NFT with 100 tokens in a single flow
2. User calls `splitTVS()` to split it 50-50 into two NFTs
3. **First iteration (i=0)** &rarr;`biddingProjects[projectId].allocations[splitNftId] = 50% * biddingProjects[projectId].allocations[splitNftId] = 50% * 100 = 50`
4. **Second iteration (i=1)**: `biddingProjects[projectId].allocations[newNftId] = 50% * biddingProjects[projectId].allocations[splitNftId] = 50% * 50 = 25`
   - Calculates 50% of 50 = 25 tokens, assigns to new NFT
5. **Result**: First NFT has 50 tokens (correct), second NFT has 25 tokens (incorrect)

User loses 25 tokens (25% of original amount) in a simple 50-50 split.

### Impact

- Users lose a significant portion of their tokens whenever they split a TVS.
- The loss is even greater when split fees are enabled.

### PoC


Assume [MissingLoopIncrement](//@TODO), [UninitializedArraysAccess](//@TODO) are fixed

```soldity
function test_poc() public {
    vm.startPrank(projectCreator);
    vesting.setVestingPeriodDivisor(1);

    // 1. Launch project
    vesting.launchBiddingProject(
        address(token),
        address(usdt),
        block.timestamp,
        block.timestamp + 1_000_000,
        "0x0",
        false
    );

    // 2. Create pool
    vesting.createPool(PROJECT_ID, 10_000 ether, 0.02 ether, true);
    vm.stopPrank();

    // 3. Place bids
    vm.deal(bidders[0], 10 ether);
    usdt.mint(bidders[0], BIDDER_USD);

    vm.startPrank(bidders[0]);
    usdt.approve(address(vesting), BIDDER_USD);
    vesting.placeBid(PROJECT_ID, BIDDER_USD, 30 days);
    vm.stopPrank();

    // 4. Prepare bid allocations
    BidInfo[] memory allBidsPool0 = new BidInfo[](2);
    allBidsPool0[0] = BidInfo({
        bidder: bidders[0],
        amount: BIDDER_USD,
        vestingPeriod: 30 days,
        poolId: 0,
        accepted: true
    });

    // 5. Generate merkle root
    bytes32[] memory poolRoots = new bytes32[](1);
    poolRoots[0] = generateMerkleProofs(allBidsPool0, 0);

    // 6. Finalize project
    vm.prank(projectCreator);
    vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);

    // 7. Bidder claim NFT
    vm.prank(bidders[0]);
    uint256 nftId = vesting.claimNFT(
        PROJECT_ID,
        0,
        BIDDER_USD,
        bidderProofs[bidders[0]]
    );

    //8. Biddder split TVS 50-50
    vm.prank(bidders[0]);
    uint256[] memory percentages = new uint256[](2);
    percentages[0] = 5000;
    percentages[1] = 5000;
    (, uint256[] memory newNftIds) = vesting.splitTVS(
        PROJECT_ID,
        percentages,
        nftId
    );

    //9. Bidder claim token (vestingPeriod passed)
    vm.warp(block.timestamp + 30 days);
    vm.startPrank(bidders[0]);
    vesting.claimTokens(PROJECT_ID, nftId);
    for (uint i = 0; i < newNftIds.length; i++) {
        vesting.claimTokens(PROJECT_ID, newNftIds[i]);
    }
    vm.stopPrank();

    uint256 bidderTokenBalance = token.balanceOf(bidders[0]);
    console.log("Expected token balance: ", BIDDER_USD);
    console.log("Actual token balance: ", bidderTokenBalance);
    console.log("Loss: ", BIDDER_USD - bidderTokenBalance);
    assertTrue(bidderTokenBalance < BIDDER_USD, "FAIL");
}
```

- Run:

```
forge clean && forge build --force
forge test --mt test_poc -vv
```

- Result:

```
[PASS] test_poc() (gas: 2867748)
Logs:
  Expected token balance:  1000000000000000000000
  Actual token balance:  750000000000000000000
  Loss:  250000000000000000000
```


### Mitigation

Iterate the loop backward (from last to first) so that the original `splitNftId` is processed last.

  