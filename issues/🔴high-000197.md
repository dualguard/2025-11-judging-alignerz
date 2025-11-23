# [000197] [High]  Legitimate allocations never contribute when the distributor token matches
  
  ### Summary

The equality guard in `protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:140-161` forces `getUnclaimedAmounts()` to return zero whenever the distributor’s configured `token` matches `vesting.allocationOf(nftId).token`. Because the distributor is instantiated precisely for that token, every legitimate allocation is filtered out, so `_setAmounts()` records `totalUnclaimedAmounts = 0` and `_setDividends()` (lines 206-219) deterministically reverts via division-by-zero. As soon as at least one NFT exists, the owner can never initialize another dividend cycle and the stablecoin reserve is frozen.

### Root Cause

 In `protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:140-161` the code checks `if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;`, which rejects every allocation whose underlying token equals the distributor’s token. The contract comments (`token` is “the token inside the TVS”) plus the constructor/setter semantics show the distributor is supposed to handle exactly that token, so the guard is inverted and filters out every legitimate NFT instead of NFTs for other tokens.

### Internal Pre-conditions

 1. Admin deploys `A26ZDividendDistributor` with `_token` equal to the vesting project’s `Allocation.token` (the documented deployment mode).
2. At least one NFT exists whose `vesting.allocationOf(id).token` equals `_token` (normal production state).
3. The owner (or anyone with `onlyOwner`) calls `setUpTheDividends()` / `_setDividends()` while the contract holds any amount of stablecoin (including 0).

### External Pre-conditions

none

### Attack Path

 1. Honest deployment: admin configures the dividend distributor for the project’s TVS token.
2. Users mint/buy an NFT producing a vesting allocation for that token.
3. Admin tries to start a dividend round via `setUpTheDividends()`.
4. `_setAmounts()` sums to zero because every `getUnclaimedAmounts()` hits the erroneous equality guard; `_setDividends()` immediately divides by zero (`unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts`) and reverts.
5. Every future attempt to set dividends repeats the same revert, so no cycle can ever be set up and the stablecoin reserve is stuck.

### Impact

 The owner can never complete `_setDividends()`, so all current and future TVS holders are permanently denied their dividend streams and the stablecoin treasury inside the distributor is frozen. This is a high-severity, permanent denial of service for the entire dividend system.

### PoC

A26ZDividendDistributorPoCTest.t.sol

```solidity
function test_PoC_matchingTokenMakesSetUpDividendsDivideByZero() public {
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 10;
        uint256[] memory claimedSeconds = new uint256[](1);
        claimedSeconds[0] = 1;
        uint256[] memory vestingPeriods = new uint256[](1);
        vestingPeriods[0] = 30 days;
        bool[] memory claimedFlows = new bool[](1);
        claimedFlows[0] = false;

        // Prime allocation with the exact token the distributor was configured with.
        vesting.primeAllocation(0, IERC20(address(tvsToken)), amounts, claimedSeconds, vestingPeriods, claimedFlows);

        vm.expectRevert(stdError.divisionError);
        distributor.setUpTheDividends();
    }

```

### Mitigation

 Invert the guard so legitimate NFTs are processed: `if (address(token) != address(alloc.token)) return 0;` (or remove the check entirely if a distributor is always single-token). Also assert `totalUnclaimedAmounts > 0` before dividing so any future accounting mistakes cannot brick the cycle silently.
  