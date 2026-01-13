# [000282] Users Don't Receive the Correct Amount of Project Tokens Causing Them To Lose Assets
  
  ### Summary

Users deposit stablecoins into a projects pool and if they win a bid they can claim the project's token over time. However, the protocol doesn't implement the token's price in the calculations, leading to user losing assets. 

### Root Cause

When an owner creates a new pool for a bidding project they have to pass as parameters the following:

- Project's ID in the `AlignerzVesting` contract
- Total allocation of tokens
- Token's price in the pool
- Bool flag for if has extra refund tokens

The issue arises as the token's price of the pool is not used anywhere in the codebase. The consequence of this means that each token that is claimed by winning bidders will simply get the amount they deposited (in stablecoin [$]) in the projects token. This is wrong as the projects token $ABD (for example) is not worth $1. 

1. In the `createPool` function we can see that the tokens price is never stored anywhere. 
2. When claiming NFT via `claimNFT` the user will input the $ amount they deposited. 
3. This will then be recorded as the amount:
```solidity
biddingProject.allocations[nftId].amounts.push(amount);
```
4. Then in the `claimTokens` that amount is used but for the project token. Which is then transferred to user.
5. In short, if Alice deposits $1000, they get $ABD 1000 back. This is a loss of funds assuming $1 > $ABD 1 (which is most likely is). 

If the price is 0.01 per project token, then for a $1000 deposit the user should get `1000 / 0.01 = 100_000` of $ABD token. 

### Internal Pre-conditions

1. Owner needs to launch a bidding project
2. Owner needs to setup a pool 
3. Bidders need to place bids
4. Owner needs to finalise bids
5. Winning bidders need to claimNFT
6. NFT owners need to claim project tokens

### External Pre-conditions

_No response_

### Attack Path

This isn't attack. This is a bug. It will happen to every user that wins a bid. 

### Impact

**Impact:** High <br>
If this happens to a user they will most likely lose a lot of their funds.

**Likelihood:** High <br>
Every user who wins bids will experience this.

### PoC

The following test can be pasted in the `AlignerzVestingProtocolTest.t.sol` test suite. It goes through the entire flow of vesting and shows that the users do get the exact amount as they deposited. The issue is that it is not worth the same. 

```solidity
    function test_UsersWhoDontClaimBeforeVestingPeriodLoseAssets() public {
        // 1. Setup the divisor
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // 2. Owner launches the bidding project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1_000_000,
            "0x0",
            true
        );

        // 3. Creating one pool for the bidding project in this case
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);

        // 4. Adding bidders to whitelist
        vesting.addUsersToWhitelist(bidders, PROJECT_ID);
        vm.stopPrank();

        // 5. Placing bids
        for (uint256 i = 0; i < bidders.length; i++) {
            vm.startPrank(bidders[i]);

            // Approve and place bid
            usdt.approve(address(vesting), BIDDER_USD);

            // Different vesting periods to test variety
            uint256 vestingPeriod = (i % 3 == 0) ? 90 days : (i % 3 == 1)
                ? 180 days
                : 365 days;

            // The first bidder is vesting for the shortest amount of time out of the other bidders

            vesting.placeBid(PROJECT_ID, BIDDER_USD, vestingPeriod);

            vm.stopPrank();
        }

        // 6. Prepare bid allocations (this would be done off-chain)
        uint256 biddersLength = bidders.length;
        BidInfo[] memory allBids = new BidInfo[](biddersLength);

        // Simulate off-chain allocation process
        for (uint256 i = 0; i < biddersLength; i++) {
            // Assign bids to pools (in reality, this would be based on some algorithm)
            bool accepted = i < 15; // last 15 bidders are accepted

            allBids[i] = BidInfo({
                bidder: bidders[i],
                amount: BIDDER_USD,
                vestingPeriod: (i % 3 == 0) ? 90 days : (i % 3 == 1)
                    ? 180 days
                    : 365 days,
                poolId: 0,
                accepted: accepted
            });
        }

        // 7. Generate merkle root
        bytes32[] memory poolRoot = new bytes32[](1); // Because only one pool was created
        poolRoot[0] = generateMerkleProofs(allBids, 0); // The pool has to be at index 0

        // 8. Generate refund proofs
        refundRoot = generateRefundProofs(allBids);

        // 9. Finalize project with merkle roots
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, refundRoot, poolRoot, 60);

        // 10. Users who one the bid can now claim NFTs
        uint256[] memory nftIds = new uint256[](15);
        for (uint256 i = 0; i < 15; i++) {
            address bidder = bidders[i];
            uint256 poolId = bidderPoolIds[bidder];

            vm.prank(bidder);
            uint256 nftId = vesting.claimNFT(
                PROJECT_ID,
                poolId,
                BIDDER_USD,
                bidderProofs[bidder]
            );
            nftIds[i] = nftId;
            bidderNFTIds[bidder] = nftId;

            // Verify NFT ownership
            assertEq(nft.ownerOf(nftId), bidder);
        }

        // 11. Fast forward time to simulate vesting period
        vm.warp(block.timestamp + 90 days + 1);

        // 12. Bidders can now claim tokens
        for (uint256 i = 0; i < 15; i++) {
            address bidder = bidders[i];

            vm.prank(bidder);
            vesting.claimTokens(PROJECT_ID, nftIds[i]);

            if (i % 3 == 0) {
                console.log(
                    "Bidder with 90 days vesting period: ",
                    token.balanceOf(bidder)
                );
            } else { // @audit this bidders have not had their NFT burned and can continue claiming
                console.log(
                    "Bidder with other vesting period  : ",
                    token.balanceOf(bidder)
                );
            }
        }

        vm.warp(block.timestamp + 60 days);
        vm.prank(bidders[0]);
        vm.expectRevert();
        vesting.claimTokens(PROJECT_ID, nftIds[0]);
    }
```

Output:
```zsh
  Bidder project token balance:  1000000000000000000000
  Bidder project token balance:  1000000000000000000000
  Bidder project token balance:  1000000000000000000000
  .
  .
```

### Mitigation

Store the tokens price and then use it to calculate the actual amount depending on the price and the length of vesting. 
  