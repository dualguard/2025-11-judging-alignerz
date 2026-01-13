# [000464] The `getTotalUnclaimedAmounts` function doesn't work when the total number of minted nft is 1, and also skips the last nft minted
  
  ### Summary

The `getTotalUnclaimedAmounts` function returns the total unclaimed amounts in all of the `Nft`'s. But there is a flaw in the way it handles the counter variable and the way `AlignerzNFT` contract works. 
The `getTotalUnclaimedAmounts()` function iterates over NFT IDs using a loop that starts from 0 and stops at < `nft.getTotalMinted()`.
However, the `AlignerzNFT` contract mints NFTs starting from `tokenId = 1`, `since _startTokenId() returns 1`:-
```solidity
function _startTokenId() internal pure virtual returns (uint256) {
        return 1;
    }
```
As a result, when only one NFT is minted (`totalMinted == 1`), the loop only checks `tokenId` 0 (which is invalid) and never checks `tokenId` 1, leading to incorrect and severely underreported accounting. And because it only checks `tokenId < nft.getTotalMinted()`, it doesn't check the last tokenId either which magnifies the issue as well.
```solidity
   function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
 @>       uint256 len = nft.getTotalMinted();
        for (uint i; i < len;) {
            (, bool isOwned) = safeOwnerOf(i);
            if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
            unchecked {
                ++i;
            }
        }
    }
```

Because this function is used to setup dividends and check total unclaimed amounts, these functionalities also suffer because of this issue.

### Root Cause

The root cause of this issue is iterating from index = 0 instead of index = 1 because NftId = 0 doesn't even exist.

### Internal Pre-conditions

N/A

### External Pre-conditions

N/A

### Attack Path

N/A

### Impact

- **Token ID 1 is never included in the total unclaimed calculation**, causing the system to incorrectly report 0 unclaimed amounts when only one NFT exists.
- **The last minted NFT is always skipped (i < len)**, leading to persistent undercounting even when multiple NFTs are minted.
- **Protocol-wide accounting becomes inconsistent**, since critical functions rely on a total that is always underreported.

**Severity - HIGH**

### PoC

- To run the PoC, import the `A26ZDividendDistributor` contract in `AlignerzVestingProtocolTest.t.sol` test suite.
- Copy paste the given test in `AlignerzVestingProtocolTest.t.sol` test suit and run the PoC with command `forge test --mt test_IncorrectIndexValueUsedPoC -vvvv`.
```solidity
function test_IncorrectIndexValueUsedPoC() public {
        vm.startPrank(projectCreator);
        A26ZDividendDistributor dividendDistributor = new A26ZDividendDistributor(
            address(vesting), address(nft), address(usdt), block.timestamp, 180 days, address(token)
        );

        address alice = makeAddr("Alice");
        address bob = makeAddr("bob");
        //deal tokens to the users, 1 thousand tokens to each
        deal(address(usdt), alice, 1000e6);
        deal(address(usdt), bob, 1000e6);
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // Setup the project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(
            address(token), address(usdt), block.timestamp, block.timestamp + 1000_000 days, "0x0", false
        );
        ///creating a pool with 3 thousand tokens and 1 usdt price per token
        vesting.createPool(0, 3000 ether, 1e6, false);
        vm.stopPrank();

        /// Place bids for the project
        vm.startPrank(alice);
        usdt.approve(address(vesting), 1000e6);
        vesting.placeBid(0, 1000e6, 90 days);
        vm.stopPrank();

        vm.startPrank(bob);
        usdt.approve(address(vesting), 1000e6);
        vesting.placeBid(0, 1000e6, 90 days);
        vm.stopPrank();

        //Create bind info to generate merkle proofs
        BidInfo[] memory allBids = new BidInfo[](2);

        /// Simulate off-chain allocation process
        // For simplicity, all bids are the same amount
        allBids[0] = BidInfo({bidder: alice, amount: 1000 ether, vestingPeriod: 90 days, poolId: 0, accepted: true});
        allBids[1] = BidInfo({bidder: bob, amount: 1000 ether, vestingPeriod: 90 days, poolId: 0, accepted: true});

        bytes32[] memory poolRoot = new bytes32[](1);
        poolRoot[0] = generateMerkleProofs(allBids, 0);

        //finalize bids
        vm.startPrank(projectCreator);
        vesting.finalizeBids(0, bytes32(0), poolRoot, 30 days);
        vm.stopPrank();

        ///only one user cliam nft, so total minted = 1
        vm.prank(alice);
        uint256 nftIdAlice = vesting.claimNFT(0, 0, 1000 ether, bidderProofs[alice]);
        assertEq(nft.ownerOf(nftIdAlice), alice);

        //check showing 0 unclaimed amounts even when the unclaimed amounts are more than 0
        assertEq(dividendDistributor.getTotalUnclaimedAmounts(), 0);
    }
```

**Logs:-**
```solidity
    ├─ [5989] A26ZDividendDistributor::getTotalUnclaimedAmounts()
    │   ├─ [1018] AlignerzNFT::getTotalMinted() [staticcall]
    │   │   └─ ← [Return] 1
    │   ├─ [1680] AlignerzNFT::extOwnerOf(0) [staticcall]
    │   │   └─ ← [Revert] OwnerQueryForNonexistentToken()
    │   └─ ← [Return] 0
    ├─ [0] VM::assertEq(0, 0) [staticcall]
    │   └─ ← [Return]
    └─ ← [Return]
```

### Mitigation

To mitigate this issue, start iteration in the `getTotalUnclaimedAmounts` function from Index = 1. Here is the revised version:-
```solidity
    function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
        uint256 len = nft.getTotalMinted();
        for (uint i = 1; i <= len;) {
            (, bool isOwned) = safeOwnerOf(i);
            if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
            unchecked {
                ++i;
            }
        }
    }
```
  