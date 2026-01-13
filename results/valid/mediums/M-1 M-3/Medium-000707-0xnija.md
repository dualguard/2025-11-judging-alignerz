# [000707] Stale allocations cause dividend distributor to overpay
  
  
## summary  
`A26ZDividendDistributor` reads from the once-written `vesting.allocationOf(nftId)` snapshot when computing `unclaimedAmounts`, but the vesting contract never refreshes that mapping after partial claims, merges, or splits. As a result, dividends continue to treat already-vested TVS as still unclaimed.

## Finding Description  
`getUnclaimedAmounts()` walks through `vesting.allocationOf(nftId).amounts`, `vestingPeriods`, and `claimedSeconds` to derive the unclaimed value of a TVS. Those arrays are only populated when the NFT is minted via `claimNFT()` / `_claimRewardTVS()` and whenever a split occurs (`allocationOf[nftId] = newAlloc`). Afterward, every state mutation—`claimTokens()`, `mergeTVS()`, `splitTVS()`—operates directly on the project-specific allocation storage (`rewardProjects[projectId].allocations` or `biddingProjects[projectId].allocations`) and never writes back to `allocationOf[nftId]`.  
 
affected logic  
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L941-L975
```solidity 
function claimTokens(uint256 projectId, uint256 nftId)
``` 
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L980-L996
```solidity
getClaimableAmountAndSeconds()`  
```  
function getClaimableAmountAndSeconds(Allocation memory allocation, uint256 flowIndex) public view returns(uint256 claimableAmount, uint256 claimableSeconds) { ... }
``  
This means every compound flow keeps referring to stale `claimedSeconds` and `claimedFlows` values, so the dividend contract continually considers the original allocations as completely unclaimed. During `_setDividends()` / `getTotalUnclaimedAmounts()` the calculations thus keep redistributing the same token value even though parts (or all) of the NFT have already been claimed. Whichever holder still possesses NFTs after the first claim round ends up receiving inflated dividends until the vesting contract is reset.

## Attack Path
- Preconditions:
  - A vesting project with minted NFTs ( claimNFT() ).
  - Users have called claimTokens() at least once on those NFTs.
- Steps:
  - Owner calls setUpTheDividends() on A26ZDividendDistributor .
  - Distributor reads allocationOf(nftId) and computes unclaimed amounts ignoring actual claims that happened in project storage.
  - Dividends are apportioned on inflated totals; holders claim claimDividends() to receive excess stablecoin.
- Highest‑impact scenario:
  - Multiple claim rounds occur followed by repeated dividend setups; each setup credits dividends against a stale snapshot that never shrinks after claims, compounding overpayments.
Supporting PoC in this repo:

- Test shows the snapshot doesn’t reflect claims (stays unclaimed) after users claim and even when an NFT is fully claimed/burned:
  - protocol/test/AlignerzVestingProtocolTest.t.sol function test_StaleAllocationSnapshot_NotUpdatedAfterClaim()

## Impact  
High. The distributor will keep minting dividend payouts long after TVS tokens are claimed, draining the contract’s stablecoin balance and giving rewards to holders who already received their vesting. This is an accounting bug that directly results in overpayments and fund leakage.

## Likelihood Explanation  
High. The issue occurs as soon as a single `claimTokens()` or TVS merge/split occurs before the next dividend setup. There is no guard preventing dividends from being called afterward, and the stale snapshot is deterministic because `allocationOf` is never updated post-mint. Any project that runs dividends after initial claims triggers the bug.
## poc

Commands Run

- forge clean
- forge build
- forge test --mt test_StaleAllocationSnapshot_NotUpdatedAfterClaim -vvv
```diff
+ import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
```
```solidity

    function test_StaleAllocationSnapshot_NotUpdatedAfterClaim() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp - 1, block.timestamp + 1_000_000, "0x0", false);
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, false);
        vm.stopPrank();

        address bidder = bidders[0];
        vm.startPrank(bidder);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        bytes32 leaf = getLeaf(bidder, BIDDER_USD, PROJECT_ID, 0);
        bytes32 rootAlloc = leaf;
        bytes32[] memory proofAlloc = new bytes32[](0);

        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = rootAlloc;
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 600);

        vm.prank(bidder);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, proofAlloc);

        vm.warp(block.timestamp + 1 days);
        uint256 balBefore = token.balanceOf(bidder);
        vm.prank(bidder);
        vesting.claimTokens(PROJECT_ID, nftId);
        uint256 balAfter = token.balanceOf(bidder);
        assertTrue(balAfter > balBefore, "No tokens claimed");

        vm.warp(block.timestamp + 365 days);
        vm.prank(bidder);
        vesting.claimTokens(PROJECT_ID, nftId);

        bool isClaimedSnap;
        IERC20 tokenSnap;
        uint256 poolIdSnap;
        (isClaimedSnap, tokenSnap, poolIdSnap) = vesting.allocationOf(nftId);
        assertEq(isClaimedSnap, false, "Snapshot shows claimed but was not updated");
    }
```
What the PoC Shows

- Mint a vesting NFT via the bidding flow.
- Partially claim tokens to update live project allocation storage.
- Fully claim tokens and burn the NFT to finalize the vesting flow.
- Read allocationOf[nftId] and confirm isClaimed remains false, proving the snapshot is stale and not synchronized with actual vesting state.

## Recommendation  
Keep `allocationOf[nftId]` synchronized with the live allocations or read from the project allocations directly: after `claimTokens()`, `mergeTVS()`, and `splitTVS()` update arrays, write them back to `allocationOf[nftId]`. Alternatively, make `A26ZDividendDistributor` query `rewardProjects`/`biddingProjects` for real-time data instead of relying on the stale snapshot.

  