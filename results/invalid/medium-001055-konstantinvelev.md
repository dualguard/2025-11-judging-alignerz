# [001055] [M] Early dividend claim before setup advances claimedSeconds and underpays users
  
  ### Summary
If a user calls `claimDividends()` before the owner runs `setUpTheDividends()` (i.e., before `_setAmounts()` and `_setDividends()` initialize per‑user dividend balances), the function computes the elapsed time and increases `claimedSeconds`, but `totalAmount` for that user is still 0. Thus, `claimableAmount = totalAmount * claimableSeconds / vestingPeriod` evaluates to 0 and transfers 0. Later, once dividends are set, subsequent claims are calculated over a shorter effective window (because `claimedSeconds` already advanced), permanently underpaying early claimants compared to users who waited for setup. So any early or accidential claims can affect user's divident.

### Code context
Claim logic advances time even when the amount is not initialized:
```205:224:protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol
function claimDividends() external {
    address user = msg.sender;
    uint256 totalAmount = dividendsOf[user].amount;
    uint256 claimedSeconds = dividendsOf[user].claimedSeconds;
    uint256 secondsPassed;
    if (block.timestamp >= vestingPeriod + startTime) {
        secondsPassed = vestingPeriod;
        dividendsOf[user].amount = 0;
        dividendsOf[user].claimedSeconds = 0;
    } else {
        secondsPassed = block.timestamp - startTime;
        dividendsOf[user].claimedSeconds += (secondsPassed - claimedSeconds);
    }
    uint256 claimableSeconds = secondsPassed - claimedSeconds;
    uint256 claimableAmount = totalAmount * claimableSeconds / vestingPeriod;
    stablecoin.safeTransfer(user, claimableAmount);
    emit dividendsClaimed(user, claimableAmount);
}
```
Per‑user `amount` is assigned only during setup:
```233:248:protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol
function _setDividends() internal {
    uint256 len = nft.getTotalMinted();
    for (uint256 i; i < len;) {
        (address owner, bool isOwned) = safeOwnerOf(i);
        if (isOwned) {
            dividendsOf[owner].amount +=
                (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
        }
        unchecked { ++i; }
    }
    emit dividendsSet();
}
```

### Impact
Permanent underpayment for early claimants: they “spend” accrued time but receive 0 at the early call, so the next claim yields less than peers who waited.

Medium severity is okay here because does not directly drain funds but harms users dividents.

### PoC (tests)
```solidity
 function test_DividendDistributor_ClaimBeforeSetup_StaleAmountLoss() public {
        // PoC: If a user claims dividends BEFORE the owner calls setUpTheDividends(),
        // the contract will advance claimedSeconds while paying 0 (stale amount),
        // which permanently reduces that user's future payouts.
        //
        // We set up 3 KOLs to avoid the distributor's off-by-one bug and ensure ids 1 and 2 are processed.
        address kolA = bidders[13];
        address kolB = bidders[14];
        address kolC = bidders[15];
        uint256 tvsAmount = 90 ether;
        uint256 vestingPeriod = 90 days;

        vm.startPrank(projectCreator);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 7 days);
        address[] memory kols = new address[](3);
        uint256[] memory amounts = new uint256[](3);
        kols[0] = kolA;
        kols[1] = kolB;
        kols[2] = kolC;
        amounts[0] = tvsAmount;
        amounts[1] = tvsAmount;
        amounts[2] = tvsAmount;
        vesting.setTVSAllocation(0, tvsAmount * 3, vestingPeriod, kols, amounts);
        vm.stopPrank();

        // All three claim TVS -> mint three NFTs (ids 1, 2, 3 with ERC721A starting at 1)
        vm.prank(kolA);
        vesting.claimRewardTVS(0); // id 1
        vm.prank(kolB);
        vesting.claimRewardTVS(0); // id 2
        vm.prank(kolC);
        vesting.claimRewardTVS(0); // id 3
        assertEq(nft.getTotalMinted(), 3);

        // Deploy distributor. We pass a token param != the vesting token so getUnclaimedAmounts doesn't early-return 0.
        A26ZDividendDistributor dist = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(usdt),
            block.timestamp,
            vestingPeriod,
            address(usdt)
        );

        // 1) kolA claims BEFORE dividends are set up.
        // Expected: pays 0 (stale amount), BUT advances claimedSeconds internally.
        vm.warp(block.timestamp + 1 days);
        uint256 balA0 = usdt.balanceOf(kolA);
        vm.prank(kolA);
        dist.claimDividends(); // pays 0, advances claimedSeconds
        uint256 balA1 = usdt.balanceOf(kolA);
        assertEq(balA1 - balA0, 0);

        // 2) Fund distributor and set up dividends.
        // The pot will be distributed over vestingPeriod, proportionally to unclaimed TVS weights.
        uint256 pot = 90 ether;
        usdt.mint(address(dist), pot);
        dist.setUpTheDividends();

        // 3) After another day, kolA and kolB claim.
        // Distributor iterates tokenIds 0..2, so ids 1 and 2 are included; id 3 is skipped (ok for this PoC).
        // Because kolA already "spent" 1 day of claimedSeconds (for 0 payout),
        // his second claim should be half of kolB's (who accrues for the full 2 days).
        vm.warp(block.timestamp + 1 days);

        uint256 balA2 = usdt.balanceOf(kolA);
        vm.prank(kolA);
        dist.claimDividends();
        uint256 payoutA = usdt.balanceOf(kolA) - balA2;

        uint256 balB2 = usdt.balanceOf(kolB);
        vm.prank(kolB);
        dist.claimDividends();
        uint256 payoutB = usdt.balanceOf(kolB) - balB2;

        // Assertion: payoutB == 2 * payoutA (2 effective days for kolB vs 1 for kolA).
        // This demonstrates the permanent loss caused by the early stale claim.
        assertEq(payoutB, payoutA * 2);
    }
```

### Recommended mitigations
The best mitigation here is to require amount before claims: `require(dividendsOf[user].amount > 0, "DividendsNotInitialized");` and do not mutate `claimedSeconds` otherwise.

  