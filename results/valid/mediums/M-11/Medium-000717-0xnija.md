# [000717] Re-running dividend setup multiplies payouts across rounds
  
  
## summary
_setDividends adds new distributions on top of previous ones without any per-round reset, allowing repeated setups to compound payouts.

## Finding Description
The A26ZDividendDistributor computes per-holder amounts proportional to `unclaimedAmountsIn[i]` and adds them to `dividendsOf[owner].amount` each run. - Owner controls dividend rounds via `setUpTheDividends()` which just calls `'_setAmounts()'` and `'_setDividends()'` without any epoch or idempotency guard.
- `'_setAmounts()'` snapshots the current `stablecoin` balance and `totalUnclaimedAmounts`, then `'_setDividends()'` iterates NFT IDs and adds each holder’s proportional amount onto `dividendsOf[owner].amount`.
- Root cause: use of `+=` with no per-round reset or gating. Nothing clears `dividendsOf[owner].amount` at the start of a new distribution, and `claimDividends()` only sets `amount = 0` when the vesting period fully ends, not per setup round.


[affected function code](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L214-L223)
```solidity
function _setDividends() internal {
    uint256 len = nft.getTotalMinted();
    for (uint i; i < len;) {
        (address owner, bool isOwned) = safeOwnerOf(i);
        if (isOwned)
            dividendsOf[owner].amount += (
                unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts
            );
        unchecked { ++i; }
    }
    emit dividendsSet();
}
```
Root cause: use of `+=` with no per-round reset or epoch gating. Highest-impact scenario: the owner calls `setUpTheDividends()` multiple times (cron, retries, or operational error). Each run re-credits the full distribution again, so holders can claim inflated dividends unrelated to new reserves or changed TVSs.

Location:
- [`protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:214–219`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L214-L223)
**Attack Path**
- Preconditions: Owner can call `setUpTheDividends()` arbitrarily; distributor holds a positive `stablecoin` balance; there are minted NFTs with non-zero `unclaimedAmountsIn`.
- Steps:
  - Owner calls `setUpTheDividends()` once, crediting `dividendsOf[user].amount` based on the current snapshot.
  - Owner calls `setUpTheDividends()` again without changing reserves; credits are added a second time.
  - A user calls `claimDividends()` after the vesting period; because the total amount was accrued twice, the payout is doubled relative to a single round.
- Highest-impact scenario:
  - Operational retries or cron jobs invoke `setUpTheDividends()` multiple times in the same period. Accumulated credits lead to users claiming more than the contract balance, draining reserves or causing transfer failures. “the owner calls `setUpTheDividends()` multiple times (cron, retries, or operational error). Each run re-credits the full distribution again, so holders can claim inflated dividends unrelated to new reserves or changed TVSs.”
- This behavior was confirmed by inspection and is not listed in known issues; it violates expected accounting invariants since credits should be tied to new reserves or explicitly rolled over by epoch.

## Impact

High. Pervasive overpayment across rounds with no limit or guard.
## poc

Test Location

- protocol/test/AlignerzVestingProtocolTest.t.sol: test_DividendsSetupAccumulates

Run

  
  - forge clean
  - forge build
  - forge test --match-test test_DividendSetupAccumulatesAcrossRounds -vvv

`import`
```diff
+ import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";

```
```solidity
    function test_DividendSetupAccumulatesAcrossRounds() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1000, bytes32(0), true);
        vesting.createPool(PROJECT_ID, 1_000_000 ether, 0.01 ether, false);
        address[] memory wl = new address[](2);
        wl[0] = bidders[0];
        wl[1] = bidders[1];
        vesting.addUsersToWhitelist(wl, PROJECT_ID);
        vm.stopPrank();

        vm.startPrank(bidders[0]);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        vm.startPrank(bidders[1]);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        BidInfo[] memory bidsArr = new BidInfo[](2);
        bidsArr[0] = BidInfo({bidder: bidders[0], amount: BIDDER_USD, vestingPeriod: 90 days, poolId: 0, accepted: true});
        bidsArr[1] = BidInfo({bidder: bidders[1], amount: BIDDER_USD, vestingPeriod: 90 days, poolId: 0, accepted: true});
        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = generateMerkleProofs(bidsArr, 0);
        refundRoot = bytes32(0);

        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, refundRoot, poolRoots, 60);

        vm.prank(bidders[0]);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[bidders[0]]);

        vm.warp(block.timestamp + 1 days);
        vm.prank(bidders[0]);
        vesting.claimTokens(PROJECT_ID, nftId);

        nft.mint(bidders[1]);

        Aligners26 otherToken = new Aligners26("Other", "OTH");
        A26ZDividendDistributor distributor = new A26ZDividendDistributor(address(vesting), address(nft), address(usdt), block.timestamp, 10 days, address(otherToken));
        usdt.mint(address(distributor), 100_000_000);

        distributor.setUpTheDividends();
        distributor.setUpTheDividends();

        usdt.mint(address(distributor), 100_000_000);

        vm.warp(block.timestamp + 10 days);
        uint256 balBefore = usdt.balanceOf(bidders[0]);
        vm.prank(bidders[0]);
        distributor.claimDividends();
        uint256 balAfter = usdt.balanceOf(bidders[0]);
        assertEq(balAfter - balBefore, 200_000_000);
    }
   
```
## Recommendation
Introduce a distribution epoch and gate per-user credits to the latest epoch, or reset per-user amounts before recomputing. One robust pattern:

```diff
+ uint256 public currentEpoch;
+ mapping(address => uint256) public lastCreditedEpoch;

function setDividends() external onlyOwner {
+   currentEpoch++;
    _setDividends();
}

function _setDividends() internal {
    uint256 len = nft.getTotalMinted();
    for (uint i; i < len;) {
        (address owner, bool isOwned) = safeOwnerOf(i);
        if (isOwned) {
+           if (lastCreditedEpoch[owner] != currentEpoch) {
+               dividendsOf[owner].amount = 0;
+               lastCreditedEpoch[owner] = currentEpoch;
+           }
            dividendsOf[owner].amount += (
                unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts
            );
        }
        unchecked { ++i; }
    }
    emit dividendsSet();
}
```





  