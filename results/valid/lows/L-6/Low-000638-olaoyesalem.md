# [000638] [Medium] Denial of Service in distributeRewardTVS due to Incorrect Loop Bounds
  
  ### Summary

Using the storage array length as the loop limit will cause a Denial of Service on partial reward distribution for the protocol administrators as the transaction will revert with an Index Out of Bounds panic when they attempt to distribute rewards to a subset of users.

### Root Cause

In `AlignerzVesting.sol` within the `distributeRewardTVS function`, the loop iterates based on `rewardProject.kolTVSAddresses.length` (the total number of unpaid users in storage) but attempts to access the kol input array at index `i`.

```solidity
function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
    uint256 len = rewardProject.kolTVSAddresses.length; // Root Cause: Uses storage length
    for (uint256 i; i < len;) {
        _claimRewardTVS(rewardProjectId, kol[i]); // Reverts if i >= kol.length
        // ...
    }
}
```

### Internal Pre-conditions

1.) Admin needs to have set up a Reward Project with multiple KOLs (e.g., rewardProject.kolTVSAddresses has length 5).

2.) Time must be past rewardProject.claimDeadline.

### External Pre-conditions

none

### Attack Path

It is time distribute rewards to a specific subset of users (e.g., 2 users out of 5) to save gas or manage batches.

The User  calls `distributeRewardTVS(projectId, [UserA, UserB])`.

The loop runs for` i=0` and` i=1` successfully.

The loop continues to `i=2` (because the storage length is 5).

The contract attempts to access k`ol[2]`. Since the input array only has length `2`, this index is invalid.

The transaction reverts with `Panic(0x32) (Array Out of Bounds)`.

### Impact

- The Protocol Admin cannot execute partial distributions. They are forced to process the entire list of KOLs in a single transaction. If the list of KOLs is large, this prevents batching and could potentially lead to a Block Gas Limit DoS if the full list becomes too expensive to process at once.

### PoC

Add this function to the test contract `AlignerzVestingProtocolTest.t.sol`

```solidity

    // ---
    // --- PoC for Out of Bounds Denial of Service in distributeRewardTVS ---
    // ---
    function test_olaoyesalem_DistributeRewardTVS_OutOfBounds_DoS() public {
        // 1. SETUP: Launch a Reward Project
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        // Claim window is 1 day
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 1 days); 
        uint256 projectId = 0;

        // 2. ALLOCATE: Add 5 KOLs to the project
        uint256 numKols = 5;
        address[] memory kols = new address[](numKols);
        uint256[] memory amounts = new uint256[](numKols);

        for (uint256 i = 0; i < numKols; i++) {
            kols[i] = bidders[i];
            amounts[i] = 100 ether;
        }

        // Approve tokens
        token.approve(address(vesting), 5000 ether); 
        vesting.setTVSAllocation(projectId, 500 ether, 90 days, kols, amounts);
        vm.stopPrank();

        // 3. : Move past the claim deadline
        vm.warp(block.timestamp + 2 days);

        // 4. EXECUTE: Try to distribute to a SUBSET of users (e.g., just 2 of them)
        address[] memory partialList = new address[](2);
        partialList[0] = bidders[0];
        partialList[1] = bidders[1];

        // 5. EXPECT REVERT (FIXED SYNTAX)
        // We expect a Panic(uint256) with code 0x32 (Array Out of Bounds)
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", 0x32));
        
        vesting.distributeRewardTVS(projectId, partialList);
    }

```

### Mitigation

- Update the loop condition to iterate based on the length of the input array (kol), not the storage array.

```solidity

function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
    // FIX: Use the input array length
    uint256 len = kol.length; 
    
    for (uint256 i; i < len;) {
        _claimRewardTVS(rewardProjectId, kol[i]);
        unchecked {
            ++i;
        }
    }
}
```
  