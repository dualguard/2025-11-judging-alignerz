# [000195] Dividend operator will permanently divert the newest holder's dividend stream to earlier NFT holders
  
  ### Summary

 The off-by-one iteration in `getTotalUnclaimedAmounts`/`_setDividends` will cause a total dividend loss for the most recently minted NFT holder as the dividend operator calls `setUpTheDividends` and loops over `[0, getTotalMinted())` even though live token IDs start at 1.

### Root Cause

 - In `protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:127-135` the loop calls `safeOwnerOf(i)` starting at `i = 0`, so token ID `getTotalMinted()` (the latest mint) is never added to `totalUnclaimedAmounts`.
- In `protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:215-223` the same `[0, len)` loop funds `dividendsOf[owner]`, again omitting the newest NFT owner and reallocating their entire share to others.

### Internal Pre-conditions

 1. A minter creates at least one NFT so that `getTotalMinted()` is `>= 1` and token IDs are `1..n` (per `protocol/src/contracts/nft/ERC721A.sol:105`).
2. The dividend operator funds the distributor with stablecoin and remains the owner so they can call `setUpTheDividends`.
3. The victim is the most recently minted NFT (no newer mint occurs between their mint and the operator calling `setUpTheDividends`).

### External Pre-conditions

None

### Attack Path

 1. Operator calls `setUpTheDividends()`, which triggers `_setAmounts()` → `getTotalUnclaimedAmounts()` and iterates `i = 0..len-1`, so the newest NFT ID (`len`) is never counted.
2. `_setDividends()` reuses the same iterator, so only prior owners receive allocations while the latest owner receives zero.
3. When holders later call `claimDividends()`, every previous owner withdraws their (inflated) share, while the most recent holder gets nothing.

### Impact

 - The newest NFT holder suffers a 100% loss of their expected dividend for each configuration round; earlier holders gain the full stolen amount.

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
import {IAlignerzNFT} from "../src/interfaces/IAlignerzNFT.sol";

/// @notice Minimal vesting stub that exposes programmable allocations.
contract VestingStub is IAlignerzVesting {
    mapping(uint256 => Allocation) private _allocations;

    function primeAllocation(
        uint256 nftId,
        IERC20 allocationToken,
        uint256[] memory amounts,
        uint256[] memory claimedSeconds,
        uint256[] memory vestingPeriods,
        bool[] memory claimedFlows
    ) external {
        require(amounts.length == claimedSeconds.length, "len mismatch");
        require(amounts.length == vestingPeriods.length, "len mismatch");
        require(amounts.length == claimedFlows.length, "len mismatch");

        Allocation storage alloc = _allocations[nftId];
        alloc.token = allocationToken;
        alloc.amounts = amounts;
        alloc.claimedSeconds = claimedSeconds;
        alloc.vestingPeriods = vestingPeriods;
        alloc.claimedFlows = claimedFlows;
    }

    function allocationOf(uint256 nftId) external view returns (Allocation memory) {
        return _allocations[nftId];
    }
}

/// @notice Minimal NFT stub that lets tests control ownership.
contract NFTStub is IAlignerzNFT {
    uint256 private _totalMinted;
    mapping(uint256 => address) private _owners;

    function mint(address to) external returns (uint256 tokenId) {
        tokenId = _totalMinted;
        _owners[tokenId] = to;
        _totalMinted++;
    }

    function extOwnerOf(uint256 tokenId) external view returns (address) {
        address owner = _owners[tokenId];
        require(owner != address(0), "NOT_MINTED");
        return owner;
    }

    function burn(uint256 tokenId) external {
        delete _owners[tokenId];
    }

    function getTotalMinted() external view returns (uint256) {
        return _totalMinted;
    }

    function setOwner(uint256 tokenId, address owner) external {
        _owners[tokenId] = owner;
        if (tokenId >= _totalMinted) {
            _totalMinted = tokenId + 1;
        }
    }
}


contract A26ZDividendDistributorLatestMintedPoCTest is Test {
    A26ZDividendDistributor internal distributor;
    VestingStub internal vesting;
    AlignerzNFT internal nft;
    MockUSD internal stablecoin;
    MockUSD internal secondaryToken;
    Aligners26 internal tvsToken;

    address internal constant holderOne = address(0xAAAA);
    address internal constant holderTwo = address(0xBBBB);

    function setUp() public {
        stablecoin = new MockUSD();
        secondaryToken = new MockUSD();
        tvsToken = new Aligners26("Aligners26", "A26Z");
        vesting = new VestingStub();
        nft = new AlignerzNFT("Alignerz", "ANFT", "ipfs://alignerz/");

        // Set up the dividend distributor with the production NFT implementation.
        distributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(stablecoin),
            block.timestamp,
            30 days,
            address(tvsToken)
        );

        // Mint two NFTs (IDs 1 and 2) to distinct holders.
        nft.mint(holderOne);
        nft.mint(holderTwo);

        // Seed the distributor with the stablecoin reserve that should be shared.
        stablecoin.transfer(address(distributor), 1_000_000);
    }

    function test_latestMintedNFTNeverReceivesDividends() public {
        _primeAllocation(1, 500_000);
        _primeAllocation(2, 500_000);

        distributor.setUpTheDividends();

        vm.warp(distributor.startTime() + distributor.vestingPeriod());

        uint256 holderOneBalanceBefore = stablecoin.balanceOf(holderOne);
        vm.prank(holderOne);
        distributor.claimDividends();
        uint256 holderOnePayout = stablecoin.balanceOf(holderOne) - holderOneBalanceBefore;
        assertGt(holderOnePayout, 0, "First minter should receive dividends");

        uint256 holderTwoBalanceBefore = stablecoin.balanceOf(holderTwo);
        vm.prank(holderTwo);
        distributor.claimDividends();
        uint256 holderTwoPayout = stablecoin.balanceOf(holderTwo) - holderTwoBalanceBefore;
        assertEq(holderTwoPayout, 0, "Latest minter is permanently skipped");

        assertEq(
            stablecoin.balanceOf(address(distributor)),
            0,
            "Entire dividend pool was assigned to the first minter"
        );
    }

    function _primeAllocation(uint256 nftId, uint256 amount) internal {
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = amount;
        uint256[] memory claimedSeconds = new uint256[](1);
        claimedSeconds[0] = 1;
        uint256[] memory vestingPeriods = new uint256[](1);
        vestingPeriods[0] = distributor.vestingPeriod();
        bool[] memory claimedFlows = new bool[](1);
        claimedFlows[0] = false;

        vesting.primeAllocation(
            nftId,
            IERC20(address(secondaryToken)),
            amounts,
            claimedSeconds,
            vestingPeriods,
            claimedFlows
        );
    }
}

```

### Mitigation

_No response_
  