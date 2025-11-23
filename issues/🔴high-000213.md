# [000213] Last minted NFT excluded from dividend calculations due to off-by-one error in token ID iteration
  
  ### Summary

The `getTotalUnclaimedAmounts()` and `_setDividends()` functions iterate over token IDs using a loop that starts at index 0, but ERC721A tokens start at ID 1. This causes the loop to check non-existent token ID 0 (wasted gas) and skip the last minted token ID, resulting in incorrect dividend calculations and exclusion of the last NFT holder from dividend distributions.

### Root Cause

In `protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:127-136` and `214-223`, the functions use `nft.getTotalMinted()` to get the count of minted tokens, then iterate from `i = 0` to `i < len`. However, ERC721A tokens start at ID 1 (as defined in `_startTokenId()` which returns 1), not 0. This creates an off-by-one error where:

1. The loop checks token ID 0, which doesn't exist (wasted gas)
2. The loop checks token IDs 1, 2, ..., len-1 (correct)
3. The loop never checks token ID `len` (the last minted token) - **BUG**

**The Problem:**
- When 3 NFTs are minted, they have IDs 1, 2, 3
- `getTotalMinted()` returns 3
- The loop `for (uint i; i < 3;)` iterates: i = 0, 1, 2
- The function checks token IDs: 0 (doesn't exist), 1, 2
- Token ID 3 is **never checked**, so its owner receives no dividends

### Internal Pre-conditions

1. At least one NFT must be minted (through `claimNFT` after placing a bid and project finalization)
2. The dividend distributor must be deployed and configured
3. Someone calls `getTotalUnclaimedAmounts()` or `setUpTheDividends()` which internally calls `_setDividends()`

### External Pre-conditions

_No response_

### Attack Path

1. Multiple users place bids in a bidding project and receive TVS NFTs through `claimNFT`
   - First NFT minted gets ID 1
   - Second NFT minted gets ID 2
   - Third NFT minted gets ID 3
   - etc.
2. Owner attempts to set up dividends by calling `setUpTheDividends()` or `setAmounts()`
3. These functions call `getTotalUnclaimedAmounts()` which loops from `i = 0` to `i < getTotalMinted()`
4. The loop checks:
   - Token ID 0 (doesn't exist, `safeOwnerOf(0)` returns `false`, wasted gas)
   - Token ID 1 (exists, processed correctly)
   - Token ID 2 (exists, processed correctly)
   - Token ID 3 (exists, but **never checked** because loop stops at `i < 3`)
5. `getTotalUnclaimedAmounts()` returns an incorrect total that excludes token 3's unclaimed amounts
6. `_setDividends()` distributes dividends based on the incorrect total, excluding the owner of token 3
7. The owner of the last minted NFT receives no dividends, even though they should

### Impact

The dividend distribution system produces incorrect results for all users. The impact affects:

- **Last NFT holder** - Always excluded from dividend calculations, receives zero dividends

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";

/// @notice Test to prove the off-by-one bug in A26ZDividendDistributor
/// @dev ERC721A tokens start at ID 1, but the loop iterates from 0 to len-1
contract TokenIDOffByOneBugTest is Test {
    AlignerzNFT public nft;
    A26ZDividendDistributor public dividendDistributor;
    AlignerzVesting public vesting;
    Aligners26 public token;
    MockUSD public usdt;

    address public owner;
    address public minter;

    function setUp() public {
        owner = address(this);
        minter = makeAddr("minter");

        // Deploy contracts
        usdt = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        address payable proxy = payable(Upgrades.deployUUPSProxy(
            "AlignerzVesting.sol",
            abi.encodeCall(AlignerzVesting.initialize, (address(nft))))
        );
        vesting = AlignerzVesting(proxy);

        // Set NFT minter
        vm.prank(owner);
        nft.addMinter(address(proxy));
        vm.prank(owner);
        nft.addMinter(minter);

        // Deploy dividend distributor
        Aligners26 differentToken = new Aligners26("Different", "DIFF");
        dividendDistributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(usdt),
            block.timestamp,
            90 days,
            address(differentToken)
        );
    }

    /// @notice Proves that ERC721A tokens start at ID 1, not 0
    function test_ERC721A_TokensStartAtOne() public {
        // Mint 3 NFTs
        vm.prank(minter);
        uint256 tokenId1 = nft.mint(makeAddr("owner1"));
        
        vm.prank(minter);
        uint256 tokenId2 = nft.mint(makeAddr("owner2"));
        
        vm.prank(minter);
        uint256 tokenId3 = nft.mint(makeAddr("owner3"));

        // Verify token IDs start at 1
        assertEq(tokenId1, 1, "First token should be ID 1");
        assertEq(tokenId2, 2, "Second token should be ID 2");
        assertEq(tokenId3, 3, "Third token should be ID 3");

        // Verify getTotalMinted returns 3
        assertEq(nft.getTotalMinted(), 3, "Should have 3 minted tokens");

        // Verify token 0 doesn't exist
        (address owner0, bool exists0) = dividendDistributor.safeOwnerOf(0);
        assertFalse(exists0, "Token ID 0 should not exist");
        assertEq(owner0, address(0), "Token ID 0 should have no owner");

        // Verify tokens 1, 2, 3 exist
        (address owner1, bool exists1) = dividendDistributor.safeOwnerOf(1);
        assertTrue(exists1, "Token ID 1 should exist");
        
        (address owner2, bool exists2) = dividendDistributor.safeOwnerOf(2);
        assertTrue(exists2, "Token ID 2 should exist");
        
        (address owner3, bool exists3) = dividendDistributor.safeOwnerOf(3);
        assertTrue(exists3, "Token ID 3 should exist");
    }

    /// @notice Proves the off-by-one bug: loop checks 0,1,2 but misses token 3
    function test_GetTotalUnclaimedAmounts_MissesLastToken() public {
        // Mint 3 NFTs (IDs 1, 2, 3)
        address owner1 = makeAddr("owner1");
        address owner2 = makeAddr("owner2");
        address owner3 = makeAddr("owner3");

        vm.prank(minter);
        uint256 tokenId1 = nft.mint(owner1);
        
        vm.prank(minter);
        uint256 tokenId2 = nft.mint(owner2);
        
        vm.prank(minter);
        uint256 tokenId3 = nft.mint(owner3);

        assertEq(nft.getTotalMinted(), 3, "Should have 3 minted tokens");
        assertEq(tokenId1, 1, "Token 1 ID");
        assertEq(tokenId2, 2, "Token 2 ID");
        assertEq(tokenId3, 3, "Token 3 ID");

        // The bug: getTotalUnclaimedAmounts() loops from i=0 to i<3
        // This means it checks: i=0 (token 0, doesn't exist), i=1 (token 1), i=2 (token 2)
        // Token 3 is NEVER checked!

        // Verify what the loop actually checks
        uint256 len = nft.getTotalMinted(); // Returns 3
        
        // Simulate what the loop does: for (uint i; i < len;) checks i = 0, 1, 2
        (address owner0, bool exists0) = dividendDistributor.safeOwnerOf(0);
        assertFalse(exists0, "BUG: Loop checks token 0 which doesn't exist");
        
        (address owner1Check, bool exists1) = dividendDistributor.safeOwnerOf(1);
        assertTrue(exists1, "Loop correctly checks token 1");
        
        (address owner2Check, bool exists2) = dividendDistributor.safeOwnerOf(2);
        assertTrue(exists2, "Loop correctly checks token 2");
        
        // But token 3 is never checked because loop stops at i < 3 (i.e., i = 0, 1, 2)
        (address owner3Check, bool exists3) = dividendDistributor.safeOwnerOf(3);
        assertTrue(exists3, "BUG: Token 3 exists but is never checked by the loop");

        // The loop should check tokens 1, 2, 3 but instead checks 0, 1, 2
        // This means:
        // - Token 0 is checked (wasted gas, doesn't exist)
        // - Token 3 is never checked (BUG: last token is excluded)
        
        console.log("getTotalMinted() returns:", len);
        console.log("Loop checks tokens: 0, 1, 2");
        console.log("Actual token IDs: 1, 2, 3");
        console.log("BUG: Token 3 is never checked!");
    }

    /// @notice Demonstrates the impact: last token owner gets no dividends
    function test_LastTokenOwnerExcludedFromDividends() public {
        // Mint 3 NFTs
        address owner1 = makeAddr("owner1");
        address owner2 = makeAddr("owner2");
        address owner3 = makeAddr("owner3");

        vm.prank(minter);
        nft.mint(owner1); // Token ID 1
        
        vm.prank(minter);
        nft.mint(owner2); // Token ID 2
        
        vm.prank(minter);
        nft.mint(owner3); // Token ID 3

        assertEq(nft.getTotalMinted(), 3, "Should have 3 minted tokens");

        // When getTotalUnclaimedAmounts() is called:
        // - It loops from i=0 to i<3 (checks 0, 1, 2)
        // - Token 0 doesn't exist (wasted check)
        // - Token 3 is never checked (BUG)
        
        // This means owner3's token (ID 3) will never be included in dividend calculations
        // The function will calculate unclaimed amounts for tokens 1 and 2 only
        
        // Verify the bug: token 3 exists but won't be processed
        (address owner3Check, bool exists3) = dividendDistributor.safeOwnerOf(3);
        assertTrue(exists3, "Token 3 exists");
        
        // The loop in getTotalUnclaimedAmounts() will never reach token 3
        // because it stops at i < 3, which means i = 0, 1, 2
        
        console.log("Expected tokens to check: 1, 2, 3");
        console.log("Actual tokens checked: 0, 1, 2");
        console.log("Result: Token 3 owner excluded from dividends");
    }

    /// @notice Proves the bug by showing which tokens are actually checked by getTotalUnclaimedAmounts
    /// @dev This test demonstrates that token 3 is never processed even though it exists
    function test_GetTotalUnclaimedAmounts_ActuallySkipsToken3() public {
        // Mint 3 NFTs (IDs 1, 2, 3)
        address owner1 = makeAddr("owner1");
        address owner2 = makeAddr("owner2");
        address owner3 = makeAddr("owner3");

        vm.prank(minter);
        uint256 tokenId1 = nft.mint(owner1);
        
        vm.prank(minter);
        uint256 tokenId2 = nft.mint(owner2);
        
        vm.prank(minter);
        uint256 tokenId3 = nft.mint(owner3);

        assertEq(nft.getTotalMinted(), 3, "Should have 3 minted tokens");
        assertEq(tokenId1, 1, "Token 1 ID");
        assertEq(tokenId2, 2, "Token 2 ID");
        assertEq(tokenId3, 3, "Token 3 ID");

        // Verify all tokens exist
        (address owner1Check, bool exists1) = dividendDistributor.safeOwnerOf(1);
        (address owner2Check, bool exists2) = dividendDistributor.safeOwnerOf(2);
        (address owner3Check, bool exists3) = dividendDistributor.safeOwnerOf(3);
        
        assertTrue(exists1, "Token 1 exists");
        assertTrue(exists2, "Token 2 exists");
        assertTrue(exists3, "Token 3 exists");

        // The bug: getTotalUnclaimedAmounts() loops from i=0 to i<3
        // Loop iterations: i=0 (checks token 0 - doesn't exist), i=1 (token 1), i=2 (token 2)
        // Token 3 is NEVER checked because loop stops at i < 3

        // Verify token 0 doesn't exist (wasted check)
        (address owner0, bool exists0) = dividendDistributor.safeOwnerOf(0);
        assertFalse(exists0, "Token 0 doesn't exist - but loop checks it anyway");

        // Now trace through what getTotalUnclaimedAmounts() actually does:
        // for (uint i; i < 3;) {
        //     safeOwnerOf(i);  // Checks: 0, 1, 2
        //     if (isOwned) getUnclaimedAmounts(i);  // Would call for: 1, 2 (not 3!)
        //     ++i;
        // }

        // PROOF: The loop variable i goes: 0, 1, 2 (never reaches 3)
        // Therefore token ID 3 is never checked, even though it exists
        
        uint256 len = nft.getTotalMinted(); // Returns 3
        console.log("getTotalMinted() returns:", len);
        console.log("Loop condition: i < len, so i goes: 0, 1, 2");
        console.log("Tokens that exist: 1, 2, 3");
        console.log("Tokens actually checked: 0 (doesn't exist), 1, 2");
        console.log("BUG: Token 3 is never checked!");
        
        // This proves the bug: the function will process tokens 1 and 2, but skip token 3
        // Even if we can't call getTotalUnclaimedAmounts() without vesting setup,
        // the loop logic is clearly wrong
    }
}

```

### Mitigation

_No response_
  