# [000741] Incomplete Pause Implementation in AlignerzNFT Allows Token Transfers While Contract Is Paused
  
  ### Summary

The missing override of the transfer function in `AlignerzNFT.sol` will allow NFT transfers to continue during paused state for token holders, as the contract fails to enforce pause restrictions on transfer operations even though natspec/documentation states otherwise

### Root Cause

`AlignerzNFT.sol` inherits `Pausable` but does not override the transfer hook in `ERC721A.sol` with the `whenNotPaused` modifier. While `mint()` and `burn()` are protected, the transfer funcions remain unprotected

```solidity
 /// @notice pauses the contract (minting and transfers)
    function pause() external virtual onlyOwner { //@> Transfers are still possible while paused due to ERC721A implementation as it is not overriden
        _pause();
    }
```

### Internal Pre-conditions

1. Owner calls `pause()` on `AlignerzNFT.sol`
2.  At least one NFT has been minted and is owned by a user

### External Pre-conditions

None

### Attack Path

1. Owner pauses the contract by calling `pause()` to freeze operations 
2. User transfers NFT by calling `transferFrom()` or `safeTransferFrom()` on their owned tokens
3. Transfer succeeds despite the contract being paused, as there is no `whenNotPaused` check on the internal transfer logic
4. Security measure bypassed: the pause mechanism fails to achieve its intended purpose of freezing all token operations

### Impact

The protocol loses the ability to freeze NFT transfers during emergencies. Token holders can continue transferring NFTs even when the contract is paused.
This contradicts the documented behavior stating that `pause()` affects minting and transfers
```solidity
/// @notice pauses the contract (minting and transfers)
```

### PoC

The following test was ran with this command: 
```bash
forge test --match-test test_FLAW_TransfersWorkWhilePaused
```
And produced these logs:
```bash
Ran 1 test for test/PoC.t.sol:AlignerzNFTTest
[PASS] test_FLAW_TransfersWorkWhilePaused() (gas: 147113)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 46.64ms (13.22ms CPU time)
```


```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Test, console} from "forge-std/Test.sol";
// Adjust the import path based on your project structure (e.g., "../src/AlignerzNFT.sol")
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol"; 


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
     * @notice This test proves that transfers are NOT paused when the contract is paused.
     * This happens because ERC721A does not automatically check paused() in _beforeTokenTransfers.
     */
    function test_FLAW_TransfersWorkWhilePaused() public {
        address recipient = makeAddr("recipient");

        // 1. Arrange: Mint a token to tokenHolder
        vm.prank(minter);
        nft.mint(tokenHolder);
        
        assertEq(nft.ownerOf(TOKEN_ID), tokenHolder, "Setup: TokenHolder should own token");

        // 2. Arrange: The owner PAUSES the contract
        vm.prank(owner);
        nft.pause();
        
        // Verify it is actually paused
        assertTrue(nft.paused(), "Contract should be paused");

        // 3. Act: tokenHolder attempts to transfer the token while paused
        vm.prank(tokenHolder);
        nft.transferFrom(tokenHolder, recipient, TOKEN_ID);

        // 4. Assert: The transfer SUCCEEDS (The Flaw)
        // If Pausable were implemented correctly for transfers, this would have reverted.
        assertEq(
            nft.ownerOf(TOKEN_ID), 
            recipient, 
            "FLAW: Transfer succeeded despite contract being paused"
        );
    }

    

}
```

### Mitigation

Override the internal transfer function with the `whenNotPaused` modifier,
  