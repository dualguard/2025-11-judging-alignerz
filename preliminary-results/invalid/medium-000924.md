# [000924] [M-2] getTotalMinted() Returns Misleading Value After Burns (Incorrect Total Supply Reporting + Misleading Analytics)
  
  ### Summary

The getTotalMinted() function reports the total number of tokens ever minted, including burned tokens, and does not decrease after burns. This causes an inflated total that can mislead users, frontends, and analytics tools about the actual circulating supply.

### Root Cause

The contract uses _totalMinted() from ERC721A to report total tokens, but _totalMinted() does not account for burned tokens. Since getTotalMinted() directly returns this value, it inaccurately represents the current supply after burns, causing misleading data for users and analytics.

Code Link : https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/AlignerzNFT.sol#L146-L148

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path



* An attacker with mint/burn access performs repeated mints and burns.
* Because `getTotalMinted()` never decreases, each cycle artificially increases the reported supply.
* Any frontend, dashboard, or off‑chain logic relying on this value receives incorrect data.
* This can mis-trigger supply caps, distort rarity or supply-based logic, or break UI/analytics assumptions.




### Impact

Users, dashboards, or marketplaces querying getTotalMinted() will see an incorrect “total supply”. After burns, the value becomes misleading, which could confuse investors, automated systems, or analytics platforms relying on this data for reporting or decision-making.


### PoC

1. Create a AlignerzNFT.t.sol file inside Test folder .
2.  Run the test using:

```solidity
forge test --match-test test_getTotalMinted_DoesNotDecreaseAfterBurn --match-path test/AlignerzNFT.t.sol -vvvv
```
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "../src/contracts/nft/AlignerzNFT.sol";

import "../src/contracts/nft/ERC721A.sol"; 


contract AlignerzNFT_TotalMinted_PoC is Test {
    AlignerzNFT nft;
    address owner = makeAddr("owner");
    address user = makeAddr("user");
    address attacker = makeAddr("attacker");

    uint256 tokenId;

    function setUp() public {
        vm.startPrank(owner);
        nft = new AlignerzNFT("Alignerz", "ALGZ", "ipfs://test/");
        nft.addMinter(attacker);
        vm.stopPrank();

        vm.prank(owner);
        tokenId = nft.mint(user);

        assertEq(nft.ownerOf(tokenId), user);
        assertEq(nft.getTotalMinted(), 1); 
    }

    function test_getTotalMinted_DoesNotDecreaseAfterBurn() public {
        console.log("Before burn:");
        console.log("  getTotalMinted():", nft.getTotalMinted()); 
        console.log("  Actual existing tokens:", nft.balanceOf(user)); 

        vm.prank(attacker);
        nft.burn(tokenId);

        vm.expectRevert();
        nft.ownerOf(tokenId);

        console.log("After burn:");
        console.log("  getTotalMinted():", nft.getTotalMinted());
        console.log("  Actual existing tokens:", nft.balanceOf(user)); 

        vm.prank(owner);
        uint256 newId = nft.mint(user);

        console.log("After minting one more:");
        console.log("  getTotalMinted():", nft.getTotalMinted()); 
        console.log("  Actual existing tokens:", nft.balanceOf(user)); 

        assertEq(nft.getTotalMinted(), 2);     
        assertEq(nft.balanceOf(user), 1);      

        console.log("");
        console.log("getTotalMinted() returns total ever minted (including burned)");
        console.log("It does NOT reflect current supply  frontends will show wrong data");
        console.log("Recommended: Implement totalSupply() = _totalMinted() - _totalBurned()");
    }
}

```

### Mitigation

To fix this, implement a totalSupply() function that subtracts burned tokens (_totalMinted() - _totalBurned()) so it accurately reflects the current circulating supply. Additionally, deprecate or rename getTotalMinted() to totalEverMinted() to make it clear that it includes all tokens ever minted, preventing misleading data for frontends, analytics, and users.

  