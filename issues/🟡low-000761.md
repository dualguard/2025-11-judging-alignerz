# [000761] `pause()` in `AlignerzNFT` only blocks mint/burn while transfers remain allowed
  
  
### Summary

The incomplete pause mechanism in `AlignerzNFT` will cause the pause feature to only stop minting and burning but not secondary transfers as pausing the contract does not actually prevent `transferFrom` calls, despite the NatSpec claiming “minting and transfers” are paused.

### Root Cause

In `AlignerzNFT.sol:80-88` the contract inherits `Pausable` and exposes `pause` / `unpause` functions that toggle the paused state, but it only uses `whenNotPaused` on `mint` and `burn` and never checks `paused()` in transfer hooks, so ERC721A transfers continue to work normally while the contract is paused.

```solidity
    /// @notice pauses the contract (minting and transfers)
>>  function pause() external virtual onlyOwner { 
        _pause();
    }

    /// @notice unpauses the contract (minting and transfers)
    function unpause() external virtual onlyOwner {
        _unpause();
    }
```

Transfers in `ERC721A` are not wrapped with `whenNotPaused` or any pause check in this contract, and there is no override of `_beforeTokenTransfers` / `_afterTokenTransfers` or `transferFrom` / `safeTransferFrom` to enforce pausing, so the pause state has no effect on moving NFTs between users. This creates a mismatch between the documented behavior (“minting and transfers” are paused) and actual behavior (only mint/burn are blocked).

### Internal Pre-conditions

1. The `AlignerzNFT` contract must be deployed and at least one NFT minted to a non-owner user, so there is something to transfer.  
2. The contract owner must call `pause()` successfully, setting the internal paused state in `Pausable`.

### External Pre-conditions

1. No external protocol or price condition is required; the issue is purely internal to the NFT contract and how it uses `Pausable`.  
2. A regular user (NFT holder) must be able to submit a `transferFrom` or `safeTransferFrom` transaction while the contract is in the paused state.

### Attack Path

1. The owner deploys `AlignerzNFT` and mints NFTs to various users (e.g. vesting certificates or TVS ownership tokens).  
2. At some point (incident response, upgrade, emergency), the owner calls `pause()`, expecting both minting and transfers to be blocked based on the NatSpec comment.  
3. An NFT holder calls `transferFrom(from, to, tokenId)` or `safeTransferFrom(from, to, tokenId)` while `paused() == true`.  
4. Because transfers are not guarded by `whenNotPaused` or any override that checks `paused()`, the transfer succeeds and the NFT moves to the new owner, despite the contract being in a paused state.  
5. The pause mechanism therefore fails to provide the intended control over secondary market movement of vesting NFTs or TVS positions.

### Impact

The protocol owner cannot rely on `pause()` to freeze the movement of TVS NFTs in emergencies, as users can continue to transfer NFTs while the contract is paused, this might break the intended safety and control guarantees of the pause feature (operational / control risk, not direct fund theft).

### PoC

Please insert below under `test` folder with a desired `test.t.sol` name and run with `forge test --mt test_pause_does_not_block_transfers -vvv`


```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";

contract NFTPauseTransferBugTest is Test {
    AlignerzNFT public nft;

    address public owner;
    address public user1;
    address public user2;

    function setUp() public {
        owner = address(this);
        user1 = makeAddr("user1");
        user2 = makeAddr("user2");

        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");


        uint256 tokenId = nft.mint(user1);
        assertEq(tokenId, 1);
    }

    function test_pause_does_not_block_transfers() public {
        nft.pause();

        vm.prank(user1);
        nft.transferFrom(user1, user2, 1);

        assertEq(nft.ownerOf(1), user2);
    }
}
```
Result:
```bash
Ran 1 test for test/NFTPauseTransferBug.t.sol:NFTPauseTransferBugTest
[PASS] test_pause_does_not_block_transfers() (gas: 105097)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 642.58µs (100.79µs CPU time)

Ran 1 test suite in 84.33ms (642.58µs CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```


### Mitigation

To align behavior with the documentation and ensure that pause genuinely blocks transfers, extend the ERC721A transfer hooks to check `paused()` and revert when paused.
  