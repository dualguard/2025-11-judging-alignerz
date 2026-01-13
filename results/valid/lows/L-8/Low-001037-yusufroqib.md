# [001037] Users can transfer NFTs during pause state bypassing emergency controls
  
  
## Summary

Missing pause check in NFT transfer operations will allow users to continue transferring TVS NFTs during emergency pause, rendering the pause mechanism ineffective for its intended purpose of halting all NFT operations during critical incidents.

## Root Cause

In `AlignerzNFT.sol`, the contract inherits from `Pausable` and implements `whenNotPaused` modifier only on `mint()` (line 106) and `burn()` (line 121), but does not override `_beforeTokenTransfers()` hook to enforce pause state on transfers. The comment on line 80 states "pauses the contract (minting and transfers)" but transfers are not actually paused.

```solidity
/// @notice pauses the contract (minting and transfers)
function pause() external virtual onlyOwner {
    _pause();
}

function mint(address to) external whenNotPaused onlyMinter returns (uint256) {
    // ✓ Respects pause
}

function burn(uint256 tokenId) external whenNotPaused onlyMinter {
    // ✓ Respects pause
}

```

## Internal Pre-conditions

1. Owner needs to call `pause()` to pause the contract during an emergency
2. At least one user needs to own TVS NFTs

## External Pre-conditions

None - this is a missing access control that always allows transfers during pause.

## Attack Path

1. Critical vulnerability discovered in vesting logic or owner decides to pause for emergency maintenance
2. Owner calls `pause()` expecting all NFT operations to halt
3. Users can still transfer their TVS NFTs using `transferFrom()` or `safeTransferFrom()`
4. NFTs continue moving between addresses during the emergency period
5. Owner's emergency pause is ineffective for preventing NFT movements

## Impact

The pause mechanism fails to prevent NFT transfers during emergencies. Users can continue transferring TVS allocations while the contract is paused, which defeats the purpose of emergency controls. This undermines the owner's ability to freeze the system during critical security incidents or maintenance periods.

**Severity:** The documented behavior (line 80-81 comment) explicitly states transfers should be paused, but the implementation does not enforce this, creating a gap between expected and actual security controls.

## PoC


**Scenario:**

1. Owner calls `pause()`:
   ```solidity
   nft.pause();
   ```

2. Owner expects all operations to stop (per documentation)

3. Alice can still transfer her NFT to Bob:
   ```solidity
   // This succeeds even though contract is paused
   nft.transferFrom(alice, bob, 1);
   ```

4. Bob now owns the TVS allocation during the emergency period

5. NFT changes hands while contract should be frozen

**Test code:**
Add this function to `test/AlignerzVestingProtocolTest.t.sol` test file and run.
```solidity
function test_PauseDoesNotAffectTransfers() public {
    // Setup: Alice gets NFT
    vm.prank(projectCreator);
    vesting.setVestingPeriodDivisor(1);

    vm.prank(projectCreator);
    vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 90 days);

    address alice = makeAddr("alice");
    address bob = makeAddr("bob");

    address[] memory kols = new address[](1);
    uint256[] memory amounts = new uint256[](1);
    kols[0] = alice;
    amounts[0] = 1000 ether;

    vm.prank(projectCreator);
    vesting.setTVSAllocation(0, 1000 ether, 90 days, kols, amounts);

    vm.prank(alice);
    vesting.claimRewardTVS(0); // Alice gets NFT #1

    console.log("Alice owns NFT #1");

    // Owner pauses the contract
    vm.prank(owner);
    nft.pause();

    console.log("Contract is now PAUSED");
    console.log("Expected: All operations including transfers should stop");
    console.log("Actual: Transfers still work");

    // Alice can still transfer during pause
    vm.prank(alice);
    nft.transferFrom(alice, bob, 1); // ❌ This should revert but doesn't

    assertEq(nft.ownerOf(1), bob, "Bob now owns NFT during pause");
    console.log("BUG: Transfer succeeded during pause state");
}
```

## Mitigation

Override `_beforeTokenTransfers()` hook in `AlignerzNFT.sol` to enforce pause state:

```solidity
/**
 * @dev Hook that enforces pause state on transfers
 * Overrides ERC721A._beforeTokenTransfers
 */
function _beforeTokenTransfers(
    address from,
    address to,
    uint256 startTokenId,
    uint256 quantity
) internal virtual override whenNotPaused {
    super._beforeTokenTransfers(from, to, startTokenId, quantity);
}
```


  