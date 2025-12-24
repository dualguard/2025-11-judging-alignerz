# [000925] [H-3] pause() Does NOT Stop Token Transfers (Misleading Security Feature + Broken Emergency Stop)
  
  ### Summary

The pause() function claims to block minting and transfers, but only mint() and burn() are actually paused. Transfers still work, misleading users and auditors about the emergency stop.

### Root Cause

The contract applies whenNotPaused only to mint() and burn(), but does not enforce it on transfer functions (transferFrom, safeTransferFrom). The pause comment falsely claims it stops transfers, creating a misleading and ineffective emergency mechanism.

Code Link : https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/AlignerzNFT.sol#L80-L83
### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path


* Owner calls `pause()` expecting transfers to stop.
* `Pausable` sets the paused flag, but **no transfer functions check it**.
* Since `transferFrom` and `safeTransferFrom` are not overridden, they **ignore the paused state**.
* Any user (or attacker) can still transfer NFTs while the contract is paused.
* This breaks the intended emergency freeze and allows attackers to move or dump tokens even during a supposed “paused” state.


### Impact


* The pause mechanism becomes **misleading and ineffective**, providing a false sense of security.
* During an emergency (hack, exploit, compromised minter), the owner **cannot stop token transfers**, making damage uncontrollable.
* Attackers can **freely move, sell, or launder NFTs** on marketplaces even while paused.
* This destroys the purpose of an emergency stop feature and can lead to **loss of user funds, market manipulation, or inability to contain ongoing attacks**.


### PoC

1. Create a AlignerzNFT.t.sol file inside Test folder .
2.  Run the test using:

```solidity
forge test --match-test test_TransfersWorkDuringPause --match-path test/AlignerzNFT.t.sol -vvvv
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "../src/contracts/nft/AlignerzNFT.sol";

import "../src/contracts/nft/ERC721A.sol"; 



contract AlignerzNFT_PauseVuln_PoC is Test {
    AlignerzNFT nft;
    address owner = makeAddr("owner");
    address user1 = makeAddr("user1");
    address user2 = makeAddr("user2");

    uint256 tokenId;

    function setUp() public {
        vm.prank(owner);
        nft = new AlignerzNFT("TestNFT", "TNFT", "ipfs://base/");

        vm.prank(owner);
        tokenId = nft.mint(user1);

        assertEq(nft.ownerOf(tokenId), user1);
    }

    function test_TransfersWorkDuringPause() public {
        vm.prank(owner);
        nft.pause();

        vm.prank(user1);
        nft.approve(user2, tokenId);

        vm.prank(user2);
        nft.transferFrom(user1, user2, tokenId);

        assertEq(nft.ownerOf(tokenId), user2);

    }
}



```

### Mitigation

To fully enforce the pause, the contract should override the transfer functions or _beforeTokenTransfers to include a whenNotPaused check. This ensures that all token movements, including transferFrom and safeTransferFrom, are blocked when the contract is paused. Implementing this change aligns the code with the function’s comment, restores the intended emergency stop functionality, and prevents users or attackers from moving NFTs while paused.
  