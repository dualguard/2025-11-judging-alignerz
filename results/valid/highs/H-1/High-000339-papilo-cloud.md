# [000339] Allocation Mutation During Split in `AlignerzVesting.sol::splitTVS` logic
  
  ### Summary

A storage overwrite inside `splitTVS` causes the source allocation to be mutated during the first split. This results in all subsequent splits using already-reduced values, permanently corrupting vesting amounts and leading to incorrect fund distribution for all users, as the caller will repeatedly call `splitTVS` and each iteration overwrites the original allocation.

### Root Cause

In the original implementation [`AlignerzVesting.sol::splitTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054), inside `splitTVS`, the loop executes:

```solidity
   function splitTVS(...) external returns (uint256, uint256[] memory) {
        // ...
        (
            uint256 feeAmount,
            uint256[] memory newAmounts
        ) = calculateFeeAndNewAmountForOneTVS(...);

        allocation.amounts = newAmounts;

        uint256 nbOfTVS = percentages.length;
        // ...

        for (uint256 i; i < nbOfTVS; ) {
            // ...
           Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
           
           // ...
           // @audit-issue: This overwrites the original allocation
           Allocation storage newAlloc = biddingProjects[projectId].allocations[nftId];
            // which now resets the allocation here
           _assignAllocation(newAlloc, alloc);
           // ...
    }
    // ...
}

    // Then after the first iteration, inside `_computeSplitArrays` function
    // Iteration 1 (i=1): new NFT minted
    Allocation memory alloc = _computeSplitArrays(
        allocation,  // allocation.amounts is now overwritten, no longer the base
        percentage,
        nbOfFlows
    );

```

This uses the live storage allocation, which has already been overwritten during iteration `i = 0`.
Thus, iteration `i = 1` computes the split based on corrupted data.

### Internal Pre-conditions

1. `splitTVS` is called with more than one percentage e.g., 50/50.
2. `allocation.amounts` contains >0 values.
3. The loop iterates more than one time (`percentages.length > 1`).

### External Pre-conditions

_No response_

### Attack Path

1. User owns an NFT and calls `splitTVS` with percentages `[5000, 5000]`.
2. Iteration 0 computes:
   original amounts = e.g., `[9950]` -> 50% = `[4975]`.
3. Allocation is overwritten to `[4975]`.
4. Iteration 1 now computes 50% of 4975, not 9950 -> `[2487.5]`.
5. Protocol now believes the user has `[4975] + [2487.5] = 7462.5` instead of 9950.
6. Permanent loss of funds caused by allocation corruption.


### Impact

All users’s vesting amounts become permanently corrupted.
Estimated loss: up to 50% or more of a user’s allocation depending on percentage combinations.
Also, this causes a bad reputation for the protocol

### PoC

This PoC is vary long

```solidity
contract SplitTVSTest is Test {
    MockAlignerzNFT public nft;
    MockERC20 public token;
    MockAlignerzVesting public vesting;
    
    address public alice = address(0x1);
    address public bob = address(0x2);
    address public treasury = address(0x999);
    
    uint256 constant PROJECT_ID = 1;
    uint256 constant YEAR = 365 days;
    uint256 constant BASIS_POINT = 10000;
    
    function setUp() public {
        nft = new MockAlignerzNFT("A26Z NFT", "A26ZN", 18);
        token = new MockERC20("A26Z", "A26Z", 18);
        vesting = new MockAlignerzVesting(address(nft), treasury);

        vesting.createBiddingProject(PROJECT_ID, address(token), address(token));

        token.mint(address(vesting), 1_000_000 * 1e18);
        
        vm.label(alice, "Alice");
        vm.label(bob, "Bob");
        vm.label(treasury, "Treasury");
    }
    
    /*//////////////////////////////////////////////////////////////
                        BASIC FUNCTIONALITY TESTS
    //////////////////////////////////////////////////////////////*/
    
    function test_SplitTVS_Basic_50_50_Split() public {        
        // Setup: Alice owns NFT with 10,000 tokens
        uint256 nftId = nft.mint(alice);
        
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 10_000 * 1e18;
        
        uint256[] memory periods = new uint256[](1);
        periods[0] = YEAR;
        
        uint256[] memory starts = new uint256[](1);
        starts[0] = block.timestamp;
        
        vesting.setAllocation(PROJECT_ID, nftId, true, amounts, periods, starts);

        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5000; // 50%
        percentages[1] = 5000; // 50%
        
        vm.prank(alice);
        (uint256 originalId, uint256[] memory newIds) = vesting.splitTVS(PROJECT_ID, percentages, nftId);
        
        console.log("Original NFT ID:", originalId);
        for (uint i = 0; i < newIds.length; i++) {
            console.log("New NFT ID", i, ":", newIds[i]);
        }
        
        // Check allocations
        (uint256[] memory amounts0,,,,,,,) = vesting.getAllocation(PROJECT_ID, originalId, true);
        (uint256[] memory amounts1,,,,,,,) = vesting.getAllocation(PROJECT_ID, newIds[0], true);
        
        console.log("\nOriginal TVS amount:", (amounts0[0] / 1e18));
        console.log("New TVS amount:", (amounts1[0] / 1e18));

        
        // After 0.5% fee: 10,000 * 0.995 = 9,950
        // Split 50/50: 9,950 / 2 = 4,975
        uint256 expectedAfterFee = 9_950 * 1e18;
        uint256 expectedEach = 4_975 * 1e18;
        
        // This will Fail, to prove my concept
        assertEq(amounts0[0], expectedEach, "Original should have 50%");
        assertEq(amounts1[0], expectedEach, "New should have 50%");
 
        
        // Check fee was transferred
        uint256 treasuryBalance = token.balanceOf(treasury);
        console.log("Treasury received fee:", treasuryBalance);
        assertEq(treasuryBalance, 50 * 1e18, "Treasury should receive 0.5% fee");
    }
}

// Inside the MockAlignerzVesting contract I created, I added some logics to help me understand how the protocol actually works

    function setAllocation(
        uint256 projectId,
        uint256 nftId,
        bool isBidding,
        uint256[] memory amounts,
        uint256[] memory vestingPeriods,
        uint256[] memory vestingStartTimes
    ) external {
        Allocation storage alloc;

        if (isBidding) {
            alloc = biddingProjectAllocations[projectId][nftId];
        } else {
            alloc = rewardProjectAllocations[projectId][nftId];
        }

        alloc.amounts = amounts;
        alloc.vestingPeriods = vestingPeriods;
        alloc.vestingStartTimes = vestingStartTimes;
        alloc.claimedSeconds = new uint256[](amounts.length);
        alloc.claimedFlows = new bool[](amounts.length);
        alloc.isClaimed = false;

        if (isBidding) {
            alloc.token = biddingProjects[projectId].token;
        } else {
            alloc.token = rewardProjects[projectId].token;
        }

        NFTBelongsToBiddingProject[nftId] = isBidding;
        allocationOf[nftId] = alloc;
    }

    // HELPER FUNCTIONS FOR TESTING
    function getAllocation(
        uint256 projectId,
        uint256 nftId,
        bool isBidding
    )
        external
        view
        returns (
            uint256[] memory amounts,
            uint256[] memory vestingPeriods,
            uint256[] memory vestingStartTimes,
            uint256[] memory claimedSeconds,
            bool[] memory claimedFlows,
            bool isClaimed,
            address token,
            uint256 assignedPoolId
        )
    {
        Allocation storage alloc;
        if (isBidding) {
            alloc = biddingProjectAllocations[projectId][nftId];
        } else {
            alloc = rewardProjectAllocations[projectId][nftId];
        }

        return (
            alloc.amounts,
            alloc.vestingPeriods,
            alloc.vestingStartTimes,
            alloc.claimedSeconds,
            alloc.claimedFlows,
            alloc.isClaimed,
            address(alloc.token),
            alloc.assignedPoolId
        );
    }
```

### Mitigation

Use a temporary memory copy `amountsToSplit` and never mutate storage until after the loop:

```solidity
        // store post-fee amounts
        uint256[] memory amountsToSplit = newAmounts;

        // Don't update the original allocation yet
        // allocation.amounts = newAmounts;

        // PER ITERATION
        tempAlloc.amounts = amountsToSplit;
```

This ensures each iteration uses the same base values, not overwritten storage.
  