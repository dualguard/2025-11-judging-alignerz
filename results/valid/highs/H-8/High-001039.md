# [001039] Owner cannot set up dividends due to infinite loop in unclaimed amount calculation
  
  ## Summary

The missing loop increment in `A26ZDividendDistributor.getUnclaimedAmounts()` will cause the owner's dividend setup operation to hang in an infinite loop whenever any user has a claimed flow or unclaimed flow with zero claimed seconds. This results in complete denial of service for the dividend distribution system, making it impossible for the owner to distribute rewards to TVS holders.
https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161
## Root Cause

In `A26ZDividendDistributor.sol:148-160`, the for loop has `continue` statements that skip the increment, causing the loop variable to never increase when certain conditions are met, resulting in an infinite loop that consumes all gas and reverts transactions.

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    // ... variable declarations ...
    uint256 len = vesting.allocationOf(nftId).amounts.length;
    for (uint i; i < len;) {
        if (claimedFlows[i]) continue;      // ❌ No increment! Loop stuck at i
        if (claimedSeconds[i] == 0) {
            amount += amounts[i];
            continue;                        // ❌ No increment! Loop stuck at i
        }
        uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
        uint256 unclaimedAmount = amounts[i] - claimedAmount;
        amount += unclaimedAmount;
        unchecked {
            ++i;                             // ✓ Only increments here
        }
    }
    unclaimedAmountsIn[nftId] = amount;
}
```

The loop condition `i < len` will always be true when the continue statements are hit, because `i` is never incremented in those code paths, causing the loop to execute indefinitely until all gas is consumed.

## Internal Pre-conditions

1. At least one TVS holder needs to have a TVS allocation with at least 1 flow
2. At least one flow needs to be either:
   - Fully claimed (`claimedFlows[i] = true`), OR
   - Never claimed (`claimedSeconds[i] = 0`)

Note: These are normal operating conditions. Users claiming their tokens naturally leads to `claimedFlows[i] = true`. New allocations start with `claimedSeconds[i] = 0`.

## External Pre-conditions

None 

## Attack Path


1. **Users receive TVS allocations and some claim their tokens over time**
   - User Alice receives NFT `#1` with 3 flows
   - Alice claims flow 0 completely → `claimedFlows[0] = true`
   - Alice partially claims flow 1 → `claimedSeconds[1] > 0`
   - Alice hasn't claimed flow 2 yet → `claimedSeconds[2] = 0`

2. **Owner wants to distribute dividends to TVS holders**
   - Owner calls `setUpTheDividends()` at line 111

3. **Function calls internal `_setAmounts()` at line 112**
   - `_setAmounts()` calls `getTotalUnclaimedAmounts()` at line 211

4. **`getTotalUnclaimedAmounts()` loops through all NFTs**
   - For each NFT, calls `getUnclaimedAmounts(i)` at line 131

5. **`getUnclaimedAmounts()` reaches Alice's NFT `#1` with 3 flows**
   ```solidity
   for (uint i; i < 3;) {
       // Iteration 1: i = 0
       if (claimedFlows[0]) continue;  // ✓ TRUE (Alice claimed flow 0)
       // ❌ BUG: No increment! i is still 0

       // Iteration 2: i = 0 (SAME!)
       if (claimedFlows[0]) continue;  // ✓ TRUE again
       // ❌ BUG: No increment! i is still 0

       // Iteration 3: i = 0 (SAME!)
       if (claimedFlows[0]) continue;  // ✓ TRUE again
       // ... continues forever until gas runs out
   }
   ```

6. **Transaction reverts with out-of-gas error**

7. **Owner cannot set up dividends - dividend distribution system is completely broken**

## Impact

Complete denial of service for the dividend distribution system. The owner cannot distribute rewards to TVS holders once any user has claimed any flow. The protocol's secondary reward mechanism becomes permanently unusable in normal operating conditions.

**Severity Justification:**
- Owner is trusted and not malicious
- However, the owner CANNOT perform their legitimate duties due to this bug
- The bug is triggered by normal user activity (claiming tokens)
- No workaround exists - dividend system is permanently broken once any claims occur
- This affects protocol functionality and breaks a core feature

## PoC
- Add this test function to `test/AlignerzVestingProtocolTest.t.sol`
- Import this contract `import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";`
- Add this state variable `A26ZDividendDistributor public dividendDistributor;`
- Then make sure all 3 bugs from `FeesManager::calculateFeeAndNewAmountForOneTVS()` has been fixed. Such as unincremented index, uninitialized `newAmounts`, and wrong `feeAmount` calculation.
```solidity
  function test_DividendDistributor_InfiniteLoop() public {
        // Setup: Create reward project and allocate TVS to users
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // Launch reward project
        vesting.launchRewardProject(
            address(token),
            address(usdt),
            block.timestamp,
            90 days
        );

        // Allocate to 3 different users 
        address alice = makeAddr("alice");
        address bob = makeAddr("bob");
        address charlie = makeAddr("charlie");

        address[] memory kols = new address[](1);
        uint256[] memory amounts = new uint256[](1);

        // Allocate to alice
        kols[0] = alice;
        amounts[0] = 1000 ether;
        vesting.setTVSAllocation(0, 1000 ether, 90 days, kols, amounts);

        // Allocate to bob
        kols[0] = bob;
        amounts[0] = 2000 ether;
        vesting.setTVSAllocation(0, 2000 ether, 90 days, kols, amounts);

        // Allocate to charlie
        kols[0] = charlie;
        amounts[0] = 3000 ether;
        vesting.setTVSAllocation(0, 3000 ether, 90 days, kols, amounts);

        vm.stopPrank();

        // Each user claims their NFT
        vm.prank(alice);
        vesting.claimRewardTVS(0); // NFT #1

        vm.prank(bob);
        vesting.claimRewardTVS(0); // NFT #2

        vm.prank(charlie);
        vesting.claimRewardTVS(0); // NFT #3

        // Bob and Charlie transfer their NFTs to Alice
        vm.prank(bob);
        nft.transferFrom(bob, alice, 2);

        vm.prank(charlie);
        nft.transferFrom(charlie, alice, 3);

        // Now merge NFTs 2 and 3 into NFT 1 to create a multi-flow NFT
        vm.startPrank(alice);
        uint256[] memory projectIds = new uint256[](2);
        projectIds[0] = 0;
        projectIds[1] = 0;

        uint256[] memory nftsToMerge = new uint256[](2);
        nftsToMerge[0] = 2;
        nftsToMerge[1] = 3;

        vesting.mergeTVS(0, 1, projectIds, nftsToMerge);
        // Now NFT #1 has 3 flows: [1000, 2000, 3000]
        vm.stopPrank();

        console.log("\n=== Setup Complete ===");
        console.log("Alice has NFT #1 with 3 flows: [1000, 2000, 3000]");
        console.log("All flows have claimedSeconds[i] = 0 (no claims yet)");

        // Deploy dividend distributor contract BEFORE any claims
        // This triggers the bug because claimedSeconds[0] == 0
        vm.prank(projectCreator);
        dividendDistributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(usdt),
            block.timestamp,
            90 days,
            address(token)
        );

        console.log("\n=== Dividend Distributor Deployed ===");

        // Owner deposits stablecoins for dividend distribution
        usdt.mint(projectCreator, 10_000 ether);
        vm.prank(projectCreator);
        usdt.transfer(address(dividendDistributor), 10_000 ether);

        console.log("Owner deposited 10,000 USDC for dividend distribution");

        // Now owner tries to set up dividends
        console.log("\n=== Owner Attempts to Set Up Dividends ===");
        console.log("Calling setUpTheDividends()...");

        // This call will revert with out-of-gas due to infinite loop
        // The bug triggers because claimedSeconds[i] == 0 hits continue without incrementing
        vm.prank(projectCreator);
        vm.expectRevert(); 
        dividendDistributor.setUpTheDividends();

        console.log("\n=== Transaction Reverted ===");
        console.log("Transaction failed due to out-of-gas (infinite loop)");
        console.log("\n=== BUG CONFIRMED ===");
        console.log("Owner CANNOT set up dividends when users have unclaimed flows");
        console.log("This is NORMAL state - fresh allocations have claimedSeconds = 0");
       
    }
```



## Mitigation

Add loop increment in all continue paths in `src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:148-160`:

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
    uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
    uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
    uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
    bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
    uint256 len = vesting.allocationOf(nftId).amounts.length;

    for (uint i; i < len;) {
        if (claimedFlows[i]) {
            unchecked {
                ++i;  // ✓ Add increment before continue
            }
            continue;
        }
        if (claimedSeconds[i] == 0) {
            amount += amounts[i];
            unchecked {
                ++i;  // ✓ Add increment before continue
            }
            continue;
        }
        uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
        uint256 unclaimedAmount = amounts[i] - claimedAmount;
        amount += unclaimedAmount;
        unchecked {
            ++i;
        }
    }
    unclaimedAmountsIn[nftId] = amount;
}
```

  