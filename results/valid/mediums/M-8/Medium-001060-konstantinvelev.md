# [001060] [M] Off-by-one tokenId iteration in Dividend Distributor skips NFTs
  
  ### Summary
`A26ZDividendDistributor.getTotalUnclaimedAmounts()` iterates NFT “indices” from 0 to `getTotalMinted() - 1` and treats the loop index as `tokenId`. With ERC721A starting tokenId at 1, this leads to checking `tokenId=0` (nonexistent) and skipping the last real token (`tokenId = startId + minted - 1`), causing inexact aggregation and unfair distribution.

### Root Cause
Basically we are running the NFTs from count `1` as it is set on initialization in the constructor(check below):
```solidity
// ...code....

constructor(string memory name_, string memory symbol_) {
        _name = name_;
        _symbol = symbol_;
        _currentIndex = _startTokenId();
    }

    // ...code....

  function _startTokenId() internal pure virtual returns (uint256) {
        return 1;
    }

    // ...code....

     function _totalMinted() internal view returns (uint256) {
        // Counter underflow is impossible as _currentIndex does not decrement,
        // and it is initialized to _startTokenId()
        unchecked {
            return _currentIndex - _startTokenId();
        }
    }
    // ...code....
```

We can observe that the function `_startTokenId()` returns 1 which is the starting point. The issue lies in that when the contract calling `getTotalUnclaimedAmounts()` is is executed `_totalMinted` which can be seen above. However we are havin the `start token = 1` and `currentIndex = 1` and the result comes as `0` which would be inefficient as our starting index for the NFTs is `1`. Another imact from this is not only the starting point and process the 0 NFT, but at the end of the loop as the condition is `i < len`. This means that we will efficiency skip the last NFT in the calculation.

- ERC721A `_startTokenId()` returns 1.
- The distributor loops `for (uint i; i < len; ++i)` and calls `safeOwnerOf(i)` / `getUnclaimedAmounts(i)` as if `i` were a valid tokenId.

Relevant code:
```
protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol
function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
    uint256 len = nft.getTotalMinted();
    for (uint i; i < len;) {
        (, bool isOwned) = safeOwnerOf(i); // i = 0 -> extOwnerOf(0) reverts (caught)
        if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
        unchecked { ++i; }
    }
}
```

### Impact
The impact should at least Medium. Incorrect dividend weights; the last minted NFT is excluded from aggregation. This underpays that holder and overpays others. Also we are going to see the 0 NFT error for the owner.

### PoC

```solidity
    function test_DividendDistributor_OffByOne_SkipsSingleNFT() public {
        // Mint exactly one NFT through reward flow
        address kol = bidders[9];
        uint256 tvsAmount = 50 ether;
        uint256 vestingPeriod = 60 days;

        // Launch reward project and allocate to single KOL
        vm.startPrank(projectCreator);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 7 days);
        address[] memory kols = new address[](1);
        kols[0] = kol;
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = tvsAmount;
        vesting.setTVSAllocation(0, tvsAmount, vestingPeriod, kols, amounts);
        vm.stopPrank();

        // Claim -> exactly one NFT minted (tokenId = 1 for ERC721A)
        vm.prank(kol);
        vesting.claimRewardTVS(0);
        assertEq(nft.getTotalMinted(), 1);
        assertEq(nft.ownerOf(1), kol);

        // Deploy distributor with token equal to vesting token to bypass ABI mismatch inside getUnclaimedAmounts
        A26ZDividendDistributor dist = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(usdt),
            block.timestamp,
            vestingPeriod,
            address(token) // equal -> getUnclaimedAmounts returns 0 early, no ABI decoding of arrays
        );

        // Call setAmounts: due to off-by-one loop starting at 0, the single NFT (id=1) is skipped,
        // so totalUnclaimedAmounts remains zero, which demonstrates the indexing bug.
        dist.setAmounts();
        assertEq(dist.getTotalUnclaimedAmounts(), 0);
        assertEq(dist.totalUnclaimedAmounts(), 0);
    }
```

```
forge test --mt test_DividendDistributor_OffByOne_SkipsSingleNFT -vv
```

### Mitigation
- Iterate tokenIds by startId and count, not by zero-based index:
```diff
function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
    uint256 len = nft.getTotalMinted();
-    for (uint i; i < len;) {
+    for (uint i = 1; i < len;) {
        (, bool isOwned) = safeOwnerOf(i); // i = 0 -> extOwnerOf(0) reverts (caught)
        if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
        unchecked { ++i; }
    }
}
```
  