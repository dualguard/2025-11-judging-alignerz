# [000200] [High]  TVS holder will siphon dividend distributions from compliant holders
  
  ### Summary

 The stale `allocationOf` mirror in `AlignerzVesting` (`protocol/src/contracts/vesting/AlignerzVesting.sol:114`, `protocol/src/contracts/vesting/AlignerzVesting.sol:571`, `protocol/src/contracts/vesting/AlignerzVesting.sol:614`, `protocol/src/contracts/vesting/AlignerzVesting.sol:887`, `protocol/src/contracts/vesting/AlignerzVesting.sol:1092`) will cause a continuous loss of dividend funds for honest holders as a claimed TVS owner will keep appearing unclaimed and draining every `_setDividends` round (`protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:126-146`).

### Root Cause

  In `protocol/src/contracts/vesting/AlignerzVesting.sol:944-976` the `claimTokens` flow mutates the canonical allocation but never reassigns the public `allocationOf` mapping, so downstream contracts read a frozen snapshot.
- In `protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:126-146` the dividend distributor trusts `vesting.allocationOf(nftId)` to compute `unclaimedAmountsIn`, so it never observes that tokens were already claimed.

### Internal Pre-conditions

 Internal Pre-conditions
1. Project owner needs to call `launchRewardProject`/`launchBiddingProject` to create a vesting program so that NFTs are minted and mirrored into `allocationOf`.
2. Dividend operator needs to deploy `A26ZDividendDistributor` pointing at the vesting contract and call `_setAmounts`/`_setDividends`, which reads `allocationOf`.


### External Pre-conditions

 1. None.

### Attack Path

 1. Bidder/KOL calls `claimRewardTVS` or `claimNFT`, receiving tvs NFT and populating `allocationOf` with `claimedSeconds = 0`.
2. Bidder calls `claimTokens` (reward or bidding project) to withdraw vested tokens; the canonical allocation marks flows claimed but `allocationOf` stays unchanged (`protocol/src/contracts/vesting/AlignerzVesting.sol:944-976`).
3. Dividend operator runs `_setAmounts`/`_setDividends`; `getUnclaimedAmounts` reads `vesting.allocationOf(nftId)` and credits the attacker with the original full amount (`protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:126-146`).
4. Every dividend epoch distributes funds to the attacker as if nothing was claimed, draining honest holders. PoC: `protocol/test/AlignerzVestingProtocolTest.t.sol:149-179`.

### Impact

 The dividend distributor misallocates the entire dividend amount toward the attacker’s NFT, causing honest TVS holders to lose their proportional share while the attacker repeatedly captures surplus stablecoins.

### PoC

```solidity
function test_PoC_allocationOfMirrorRemainsStaleAfterClaims() public {
        address kol = bidders[0];
        uint256 rewardProjectId = vesting.rewardProjectCount();
        uint256 vestingPeriod = 30 days;
        uint256 claimWindow = 60 days;
        uint256 startTime = block.timestamp;
        uint256 allocationAmount = 1_000 ether;

        vm.startPrank(projectCreator);
        vesting.launchRewardProject(address(token), address(usdt), startTime, claimWindow);
        address[] memory recipients = new address[](1);
        recipients[0] = kol;
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = allocationAmount;
        vesting.setTVSAllocation(rewardProjectId, allocationAmount, vestingPeriod, recipients, amounts);
        vm.stopPrank();

        vm.prank(kol);
        vesting.claimRewardTVS(rewardProjectId);
        uint256 nftId = nft.getTotalMinted();

        vm.warp(startTime + vestingPeriod + 1);

        vm.prank(kol);
        vesting.claimTokens(rewardProjectId, nftId);

        assertEq(token.balanceOf(kol), allocationAmount, "claim should transfer full amount");

        (bool isClaimed,,) = vesting.allocationOf(nftId);
        assertFalse(isClaimed, "mirror never flips isClaimed");
    }

```

### Mitigation

Mirror the live allocation into `allocationOf[nftId]` whenever `claimedSeconds`, `claimedFlows`, or `isClaimed` change (claims, merges, splits, burns) or remove the redundant mapping and expose the canonical allocation through a view function used by the dividend distributor.
  