# [000162] Dividend distribution functions on will DoS due to infinite loop
  
  ### Summary

Infinite loop causes DoS in the dividend distribution

### Root Cause

The loop inside `getUnclaimedAmounts` continues when processing claimed flows and claimed seconds without changing the iterator. As a consequence, it will loop forever and the call will DoS due to insufficient gas, preventing the admin from calling them under these scenarios.

### Internal Pre-conditions

An user must have claimed at least one vesting schedule already.

### External Pre-conditions

_No response_

### Attack Path

An user has to simply claim their vesting tokens

### Impact

Dividend distribution is permanently DoSed.

### PoC
Add `import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";` and copy paste the function below in AlignerzVestingProtocolTest.t.sol
```solidity
function test_DividendDistribution_GasDoS() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        //launch project, create a pool, whitelist 2 users
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);

        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);

        address user1 = bidders[0];
        address user2 = bidders[1];
        address[] memory twoBidders = new address[](2);
        twoBidders[0] = user1;
        twoBidders[1] = user2;
        vesting.addUsersToWhitelist(twoBidders, PROJECT_ID);
        vm.stopPrank();

        vm.startPrank(user1);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        vm.startPrank(user2);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        BidInfo[] memory allBids = new BidInfo[](2);
        allBids[0] = BidInfo({
            bidder: user1,
            amount: BIDDER_USD,
            vestingPeriod: 90 days,
            poolId: 0,
            accepted: true
        });
        allBids[1] = BidInfo({
            bidder: user2,
            amount: BIDDER_USD,
            vestingPeriod: 90 days,
            poolId: 0,
            accepted: true
        });

        //generate merkle root and finalize project
        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = generateMerkleProofs(allBids, 0);

        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);

        //claim NFTs
        vm.prank(user1);
        uint256 nftId1 = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[user1]);
        assertEq(nft.ownerOf(nftId1), user1);

        vm.prank(user2);
        uint256 nftId2 = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[user2]);
        assertEq(nft.ownerOf(nftId2), user2);

        //fast forward time to fully vest tokens
        vm.warp(block.timestamp + 45 days);

        //uer1 calls claimTokens. This sets claimedFlows[i] to true for their allocation.
        vm.prank(user1);
        vesting.claimTokens(PROJECT_ID, nftId1);

        //deploy A26ZDividendDistributor
        vm.prank(owner);
        A26ZDividendDistributor distributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(usdt),
            block.timestamp,
            90 days,
            address(token) // Initially the same token
        );

        //create a different token to bypass the early return in getUnclaimedAmounts
        Aligners26 token2 = new Aligners26("Another Token", "ATK");
        distributor.setToken(address(token2));

        //fund the distributor with stablecoins for dividends
        usdt.mint(address(distributor), 1_000_000 * 1e6); // 1M USDT

        vm.prank(owner);
        vm.expectRevert(bytes("")); //out of gas
        distributor.setAmounts();
    }
```

### Mitigation

Remove the unchecked line and replace it with the loop header below. More recent solidity versions are already optimized towards gas efficient loops.
```solidity
for (uint i; i < len;++i) {
```
  