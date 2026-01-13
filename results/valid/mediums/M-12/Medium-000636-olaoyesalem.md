# [000636] [ Medium ] Accidental Lock on Claimable Funds Due to Premature Revert
  
  ### Summary

The premature `require(claimableAmount > 0)` check inside `getClaimableAmountAndSeconds` will cause a Denial of Service (DoS) on token withdrawal for users holding multi-flow vesting NFTs as the user will be blocked from claiming already accrued tokens if another flow has zero accrual.

### Root Cause

In `AlignerzVesting.sol` within the `getClaimableAmountAndSeconds` function, the check `require(claimableAmount > 0, No_Claimable_Tokens())`; forces every flow checked to yield an amount greater than zero. This fails the "allow non-accruing flows to be processed" logic.



### Internal Pre-conditions

- A user needs to own a multi-flow NFT (TVS) created via mergeTVS.
- 
The multi-flow NFT must contain at least two flows:Flow A: Vested (accrued tokens, $claimableAmount > 0$).Flow B: Not yet started or fully claimed (zero accrual, $claimableAmount = 0$).

- The caller must attempt to call claimTokens() before Flow B's vesting period begins.

### External Pre-conditions

None

### Attack Path

- Attack Path (Vulnerability Path)Setup: A user's vesting NFT (NFT #123) has been set up (e.g., via mergeTVS) with Flow A (vesting started 6 months ago) and Flow B (vesting starts 1 year from now).
- Action: The User calls `vesting.claimTokens(projectId, 123) `to withdraw tokens from Flow A.Failure: The internal loop iterates to Flow B.Revert: `getClaimableAmountAndSeconds` calculates $claimableAmount = 0$ for Flow B and hits the `require(claimableAmount > 0) `guard.

- The entire transaction reverts, blocking the user from claiming the substantial amount of tokens already accrued in Flow A.

### Impact

The users suffer a Denial of Service on their accrued tokens. They cannot access funds they have already earned (Flow A) until the flow with zero accrual (Flow B) naturally begins its vesting period, which may be months or years later. This creates an accidental, indefinite lock.

### PoC

This function  should be added  to the `AlignerzVestingProtocolTest.t.sol` Test Contract.

```solidity

// ---
   
    // ---
    // This test proves that the claimTokens function reverts if any single flow
    // has not yet started vesting, blocking access to accrued tokens in other flows.
    function test_ZeroAccrual_Blocks_Claimable_Flows() public {
        address alice = makeAddr("alice");
        address bob = makeAddr("bob"); // Dummy user
        uint256 rewardAmount = 100 ether;
        
        // 1. Setup Projects with Drastically Different Start Times
        vm.startPrank(projectCreator);
        
        // Project 0 (Flow A): Starts NOW (Claimable Flow)
        // vestingPeriod=30 days
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 100 days);
        uint256 projId0 = 0;
        
        // Project 1 (Flow B): Starts IN THE FUTURE (Non-Claimable Flow)
        uint256 futureStartTime = block.timestamp + 365 days;
        vesting.launchRewardProject(address(token), address(usdt), futureStartTime, 100 days);
        uint256 projId1 = 1;
        
        // Allocate rewards (Admin side setup)
        address[] memory singleKol = new address[](1); singleKol[0] = alice;
        uint256[] memory singleAmount = new uint256[](1); singleAmount[0] = rewardAmount;
        
        vesting.setTVSAllocation(projId0, rewardAmount, 30 days, singleKol, singleAmount);
        vesting.setTVSAllocation(projId1, rewardAmount, 30 days, singleKol, singleAmount);
        
        // Set merge fee to 0 for a clean PoC
        vesting.setMergeFeeRate(0);
        vm.stopPrank();

        // 2. Alice Claims and Merges to create a Single Multi-Flow NFT
        vm.startPrank(alice);
        
        // Claim NFT 0 (Flow A)
        vesting.claimRewardTVS(projId0); 
        uint256 nftId0 = nft.getTotalMinted(); 

        // Claim NFT 1 (Flow B)
        vesting.claimRewardTVS(projId1); 
        uint256 nftId1 = nft.getTotalMinted();
        
        // Merge NFT 1 (Future Flow) into NFT 0 (Current Flow)
        uint256[] memory projIds = new uint256[](1); projIds[0] = projId1;
        uint256[] memory nftIds = new uint256[](1); nftIds[0] = nftId1;
        vesting.mergeTVS(projId0, nftId0, projIds, nftIds);
        
        // 3. Advance time just enough for Flow A to accrue tokens (1 day)
        vm.warp(block.timestamp + 1 days); 
        
        // Sanity Check: Alice's balance must be 0 before claim
        assertEq(token.balanceOf(alice), 0, "Initial token balance must be zero");

        // 4. Trigger the Bug
        // Flow A has accrued tokens (claimableAmount > 0).
        // Flow B has not started (claimableAmount == 0).
        // The loop will hit Flow B's check and revert.
        
        // Expect Revert: No_Claimable_Tokens is the error thrown by the failing internal require
        vm.expectRevert();
        vesting.claimTokens(projId0, nftId0);
        
        vm.stopPrank();

        // If the bug was fixed, the transaction would succeed, and Alice would claim the accrued amount.
    }

```

### Mitigation

- Remove require(claimableAmount > 0, No_Claimable_Tokens()); from getClaimableAmountAndSeconds.

- Add a conditional check in claimTokens before processing the flow:

```solidity

// AlignerzVesting.sol::claimTokens
// ...
(uint256 claimableAmount, uint256 claimableSeconds) = getClaimableAmountAndSeconds(allocation, i);

if (claimableAmount > 0) {
    allocation.claimedSeconds[i] += claimableSeconds;
    // ... rest of state updates ...
    claimableAmounts += claimableAmount;
} 
// ...
// Final check at the end:
require(claimableAmounts > 0, No_Claimable_Tokens());
```
  