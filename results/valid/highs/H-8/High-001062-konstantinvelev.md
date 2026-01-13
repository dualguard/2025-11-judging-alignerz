# [001062] [H] DoS via infinite loop in A26ZDividendDistributor.getUnclaimedAmounts
  
  ### Summary
Calling `A26ZDividendDistributor.setAmounts()` or `setUpTheDividends()` can revert or run out of gas due to an infinite loop in `getUnclaimedAmounts(uint256)`. The loop uses `continue` without incrementing the loop index, causing a denial of service against dividend setup/distribution whenever there is at least one fresh or fully-claimed TVS flow. Skipping the incrementation of `i` is like looping same index over and over again, which is the exactly the reason for the issue.

### Root Cause
In `getUnclaimedAmounts(uint256 nftId)`, the loop over flows fails to increment `i` before `continue`:

```
protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol
for (uint i; i < len;) {
    if (claimedFlows[i]) continue;                 // ← i not incremented
    if (claimedSeconds[i] == 0) {
        amount += amounts[i];
        continue;                                  // ← i not incremented
    }
    uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
    uint256 unclaimedAmount = amounts[i] - claimedAmount;
    amount += unclaimedAmount;
    unchecked { ++i; }
}
```

The function is transitively invoked by:
- `getTotalUnclaimedAmounts()` which iterates all minted NFTs, and
- `_setAmounts()` which calls `getTotalUnclaimedAmounts()` inside `setAmounts()` and `setUpTheDividends()`.

### Internal Pre-conditions
- At least one NFT (TVS) exists with valid arrays `amounts`, `claimedSeconds`, `claimedFlows`, `vestingPeriods`.
- For a processed flow index `i` of that TVS, either:
  - `claimedSeconds[i] == 0` (typical right after `claimNFT`), or
  - `claimedFlows[i] == true` (flow fully claimed).
- The distributor is configured so `getUnclaimedAmounts` does not short-circuit to 0 (e.g., `token` differs from `vesting.allocationOf(nftId).token`).

### External Pre-conditions
NO

### Attack Path
1. Users bid and claim NFT certificates so that for at least one TVS, a flow has `claimedSeconds[i] == 0` (fresh allocation) or `claimedFlows[i] == true`.
2. Admin deploys/configures `A26ZDividendDistributor` and calls `setAmounts()` (or `setUpTheDividends()`).
3. `setAmounts()` → `_setAmounts()` → `getTotalUnclaimedAmounts()` → `getUnclaimedAmounts(nftId)` enters an infinite loop due to `continue` without `i++`, reverting or exhausting gas.

### Impact
- High severity DoS against dividends lifecycle:
  - `setAmounts()`/`setUpTheDividends()` cannot complete, blocking initialization/update of dividends for all holders.
  - Potential immobilization of funds earmarked for distribution.

### PoC

Test added: `test_DividendDistributor_DoS_PoC_simple` in `protocol/test/AlignerzVestingProtocolTest.t.sol`.

```solidity
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import "forge-std/console.sol";

...code...

    function test_DividendDistributor_DoS_PoC_simple() public {
        // 1) Launch minimal project to mint exactly one NFT (TVS) with claimedSeconds[0] == 0
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 10, "0x0", true);
        vesting.createPool(PROJECT_ID, 1_000_000 ether, 0.01 ether, true);
        vesting.addUsersToWhitelist(bidders, PROJECT_ID);
        vm.stopPrank();

        address bidder0 = bidders[0];
        address bidder1 = bidders[1];
        // Two bidders place bids so we can build a 2-leaf merkle
        vm.startPrank(bidder0);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();
        vm.startPrank(bidder1);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        // Build merkle root and proofs for two accepted bids in pool 0
        BidInfo[] memory accepted = new BidInfo[](2);
        accepted[0] = BidInfo({
            bidder: bidder0,
            amount: BIDDER_USD,
            vestingPeriod: 90 days,
            poolId: 0,
            accepted: true
        });
        accepted[1] = BidInfo({
            bidder: bidder1,
            amount: BIDDER_USD,
            vestingPeriod: 90 days,
            poolId: 0,
            accepted: true
        });
        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = generateMerkleProofs(accepted, 0);

        // Finalize and allow claiming NFT
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);

        // Both bidders claim to mint two NFTs (IDs 1 and 2 with ERC721A start at 1).
        // This ensures getTotalMinted() == 2 and the loop will process tokenId 1,
        // which has claimedSeconds[0] == 0 and will hit the buggy `continue;`.
        vm.prank(bidder0);
        uint256 nftId1 = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[bidder0]);
        assertEq(nft.ownerOf(nftId1), bidder0);
        vm.prank(bidder1);
        uint256 nftId2 = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[bidder1]);
        assertEq(nft.ownerOf(nftId2), bidder1);

        // 2) Deploy DividendDistributor with a different token than the vesting token
        //    to avoid early return and hit the buggy loop.
        Aligners26 otherToken = new Aligners26("Other", "OTH");
        A26ZDividendDistributor dist = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(usdt),
            block.timestamp,
            90 days,
            address(otherToken)
        );

        // 3) setAmounts() -> calls getTotalUnclaimedAmounts() -> getUnclaimedAmounts(1)
        // no matters how much gas we give, it will revert due to the infinite loop always.
        vm.expectRevert();
        dist.setAmounts{gas: 100000000}();
    }
```

Command:
```
forge test --mt test_DividendDistributor_DoS_PoC_simple -vv
```

### Mitigation
- Increment the loop index before each `continue` and at the end of the iteration:

```
for (uint i; i < len;) {
    if (claimedFlows[i]) { unchecked { ++i; } continue; }
    if (claimedSeconds[i] == 0) {
        amount += amounts[i];
        unchecked { ++i; }
        continue;
    }
    uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
    uint256 unclaimedAmount = amounts[i] - claimedAmount;
    amount += unclaimedAmount;
    unchecked { ++i; }
}
```

Additional hardening:
- Prefer `for (uint i = 0; i < len; ++i)` to avoid manual index management pitfalls.

  