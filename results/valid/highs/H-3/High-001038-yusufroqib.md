# [001038] User will be unable to split TVS due to out-of-bounds array access in `AlignerzVesting::_computeSplitArrays`
  
  ## Summary

Uninitialized memory arrays in `_computeSplitArrays()` will cause complete denial of service for users attempting to split TVS as the function will revert with out-of-bounds errors when accessing array indices.
https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113-L1141

## Root Cause

In `AlignerzVesting.sol:1134-1143`, the `_computeSplitArrays()` function attempts to write to memory array fields of the `alloc` struct without initializing them first. Memory arrays in Solidity default to length 0, causing out-of-bounds access on the first write attempt.

```solidity
  function _computeSplitArrays(
        Allocation storage allocation,
        uint256 percentage,
        uint256 nbOfFlows
    )
        internal
        view
        returns (
            Allocation memory alloc
        )
    {
        uint256[] memory baseAmounts = allocation.amounts;
        uint256[] memory baseVestings = allocation.vestingPeriods;
        uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
        uint256[] memory baseClaimed = allocation.claimedSeconds;
        bool[] memory baseClaimedFlows = allocation.claimedFlows;
        alloc.assignedPoolId = allocation.assignedPoolId;
        alloc.token = allocation.token;
        for (uint256 j; j < nbOfFlows;) {
      // ❌ OUT OF BOUNDS: array has length 0
            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
            alloc.vestingPeriods[j] = baseVestings[j];
            alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
            alloc.claimedSeconds[j] = baseClaimed[j];
            alloc.claimedFlows[j] = baseClaimedFlows[j];
            unchecked {
                ++j;
            }
        }
    }
```

## Internal Pre-conditions

1. User needs to own at least one TVS NFT with at least 1 flow
2. User needs to call `splitTVS()` with valid split percentages

## External Pre-conditions

None 

## Attack Path

1. User calls `splitTVS()` to divide their TVS allocation
2. Function calls `_computeSplitArrays()` at line 1077
3. Loop attempts to write to `alloc.amounts[0]` at line 1135
4. Transaction reverts with `panic: array out-of-bounds access (0x32)`
5. User cannot split their TVS allocation

## Impact

Users cannot split their TVS allocations. The `splitTVS()` function is completely non-functional, making it impossible to divide vesting allocations into smaller portions. This breaks a core protocol feature.

## PoC

**Setup:**
- Alice has NFT #1 with 1 flow of 1000 tokens
- Alice wants to split it 50/50 into two NFTs

**Alice attempts split:**

1. Alice calls `splitTVS(0, [5000, 5000], 1)`

2. Function reaches `_computeSplitArrays()` at line 1077:
   ```solidity
   Allocation memory firstAlloc = _computeSplitArrays(allocation, 5000, 1);
   ```

3. Inside `_computeSplitArrays()`:
   ```solidity
   Allocation memory alloc;  // All array fields default to length 0

   for (uint256 j; j < 1;) {
       alloc.amounts[j] = (baseAmounts[0] * 5000) / 10000;  // ❌ OUT OF BOUNDS
       // Trying to write to index 0 of array with length 0
   }
   ```

4. Transaction reverts with `panic: array out-of-bounds access (0x32)`

5. Alice cannot split her NFT

- Add this test to `test/AlignerzVestingProtocolTest.t.sol`

```solidity
  function test_UninitializedSplitArrays_OutOfBounds() public {
        // Setup: Create reward project and allocate TVS
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // Launch reward project
        vesting.launchRewardProject(
            address(token),
            address(usdt),
            block.timestamp,
            90 days
        );

        // Allocate to Alice
        address alice = makeAddr("alice");
        address[] memory kols = new address[](1);
        uint256[] memory amounts = new uint256[](1);

        kols[0] = alice;
        amounts[0] = 1000 ether;
        vesting.setTVSAllocation(0, 1000 ether, 90 days, kols, amounts);

        vm.stopPrank();

        // Alice claims her NFT
        vm.prank(alice);
        vesting.claimRewardTVS(0); // NFT #1

        console.log("\n=== Setup Complete ===");
        console.log("Alice has NFT #1 with 1 flow: 1000 tokens");
        console.log("Alice wants to split it 50/50 into 2 NFTs");

        // Alice attempts to split the NFT
        console.log("\n=== Alice Attempts to Split NFT ===");
     

        vm.startPrank(alice);

        uint256[] memory splitPercentages = new uint256[](2);
        splitPercentages[0] = 5000; // 50%
        splitPercentages[1] = 5000; // 50%

        // This will revert with panic: array out-of-bounds access (0x32)
        vm.expectRevert(stdError.indexOOBError);
        vesting.splitTVS(0, splitPercentages, 1);

        vm.stopPrank();

    }
```

## Mitigation

Initialize all memory arrays in `_computeSplitArrays()` before the loop:

```solidity
function _computeSplitArrays(
    Allocation storage allocation,
    uint256 percentage,
    uint256 nbOfFlows
) internal view returns (Allocation memory alloc) {
    uint256[] memory baseAmounts = allocation.amounts;
    uint256[] memory baseVestings = allocation.vestingPeriods;
    uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
    uint256[] memory baseClaimed = allocation.claimedSeconds;
    bool[] memory baseClaimedFlows = allocation.claimedFlows;

    // ✓ Initialize arrays with correct length
    alloc.amounts = new uint256[](nbOfFlows);
    alloc.vestingPeriods = new uint256[](nbOfFlows);
    alloc.vestingStartTimes = new uint256[](nbOfFlows);
    alloc.claimedSeconds = new uint256[](nbOfFlows);
    alloc.claimedFlows = new bool[](nbOfFlows);

    alloc.assignedPoolId = allocation.assignedPoolId;
    alloc.token = allocation.token;

    for (uint256 j; j < nbOfFlows;) {
        alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
        alloc.vestingPeriods[j] = baseVestings[j];
        alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
        alloc.claimedSeconds[j] = baseClaimed[j];
        alloc.claimedFlows[j] = baseClaimedFlows[j];
        unchecked {
            ++j;
        }
    }
}
```

  