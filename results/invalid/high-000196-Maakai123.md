# [000196] NFT sellers will keep all dividends owed to NFT buyers
  
  ### Summary

 Address-based dividend snapshots will cause a total loss of promised dividends for current TVS holders as the prior NFT owner will transfer the NFT and continue claiming the entire stream.

### Root Cause

 In `protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:45-219` the contract stores dividends in `dividendsOf[address]` during `_setDividends` and `claimDividends` later authorizes withdrawals solely by `msg.sender`, so entries are never re-bound to the NFT’s new owner.

### Internal Pre-conditions

 1. Admin needs to fund the distributor and call `setUpTheDividends()` so that `dividendsOf[seller].amount` becomes greater than zero (`protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:111-214`).
2. The seller needs to transfer their NFT to a buyer using the unrestricted ERC721 transfer flow (`protocol/src/contracts/nft/AlignerzNFT.sol:82-134`).


### External Pre-conditions

 None

### Attack Path

 1. Seller A holds NFT  number 1  when `setUpTheDividends()` runs, populating `dividendsOf[A]`.
2. A transfers NFT  number 1  to buyer B via the standard ERC721 path.
3. B calls `claimDividends()` but `dividendsOf[B]` is zero, so no stablecoin is paid.
4. A continues calling `claimDividends()` on their address and drains the entire dividend stream even though A no longer owns the NFT.

### Impact

 The actual TVS holders (current NFT owners) suffer a 100% loss of their dividend stream because sellers irrevocably keep the stablecoin payouts after selling the NFT; attackers pocket both the NFT sale proceeds and all dividends.

### PoC

```solidity
   function test_PoC_dividendsDoNotFollowNFTTransfer() public {
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 100;
        uint256[] memory claimedSeconds = new uint256[](1);
        claimedSeconds[0] = 1;
        uint256[] memory vestingPeriods = new uint256[](1);
        vestingPeriods[0] = 30 days;
        bool[] memory claimedFlows = new bool[](1);
        claimedFlows[0] = false;

        vesting.primeAllocation(0, IERC20(address(secondaryToken)), amounts, claimedSeconds, vestingPeriods, claimedFlows);
        distributor.setUpTheDividends();

        address buyer = address(0xCAFE);
        nft.setOwner(0, buyer);

        uint256 startTime = distributor.startTime();
        vm.warp(startTime + 15 days);

        uint256 buyerBalanceBefore = stablecoin.balanceOf(buyer);
        vm.prank(buyer);
        distributor.claimDividends();
        assertEq(
            stablecoin.balanceOf(buyer) - buyerBalanceBefore,
            0,
            "New NFT owner receives no dividends because mapping never updated"
        );

        uint256 sellerBalanceBefore = stablecoin.balanceOf(nftHolder);
        vm.prank(nftHolder);
        distributor.claimDividends();
        assertGt(
            stablecoin.balanceOf(nftHolder) - sellerBalanceBefore,
            0,
            "Previous owner keeps the dividend stream after selling"
        );
    }
```

### Mitigation

 rack dividends per NFT ID and require the caller of `claimDividends(tokenId)` to be `nft.extOwnerOf(tokenId)`, or hook into NFT transfers to reassign the seller’s `dividendsOf` entry to the new owner so dividend rights always follow the token.

  