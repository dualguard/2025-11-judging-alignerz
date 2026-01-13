# [000198] [High]    Users can permanently DoS dividend distribution by leaving any vesting flow fresh or marked claimed
  
  ### Summary

 The missing index increment in `protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:147-157` will cause a total dividend setup failure for the protocol as any user holding an NFT with a fresh or fully-claimed vesting flow can trigger `_setAmounts() → getTotalUnclaimedAmounts() → getUnclaimedAmounts()` to loop forever and revert.

### Root Cause

 In `protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:147-157` the loop lacks increment statements inside the branches guarded by `claimedFlows[i]` and `claimedSeconds[i] == 0`, so invoking `continue` preserves `i` and the loop never terminates for those everyday states.

### Internal Pre-conditions

1. NFT holder needs to keep at least one vesting flow where `claimedSeconds` remains 0 (never claimed) or `claimedFlows` is true (fully claimed).
2. Owner needs to call `setUpTheDividends()` or `setAmounts()` so `_setAmounts()` is executed and iterates over that NFT.

### External Pre-conditions

 None. The state arises from normal vesting behavior without external protocol manipulation.

### Attack Path

 1. A regular NFT holder maintains or creates an allocation with `claimedSeconds == 0` (default) or finishes a flow so the vesting contract sets `claimedFlows == true`.
2. The owner attempts to refresh dividend accounting by calling `setUpTheDividends()` (or `setAmounts()`), which in turn calls `_setAmounts()`.
3. `_setAmounts()` calls `getTotalUnclaimedAmounts()`, which loops over NFTs and invokes `getUnclaimedAmounts()` on the attacker’s token.
4. `getUnclaimedAmounts()` hits the problematic branch, never increments `i`, and consumes all gas; the entire transaction reverts.

### Impact

 The protocol cannot execute `setUpTheDividends()` / `setAmounts()` / `setDividends()`, meaning no new dividend rounds can be initialized and stablecoin deposits remain undistributed indefinitely. This is a full liveness failure for dividend distribution with no attacker profit (griefing)

### PoC

protocol/test/A26ZDividendDistributorPoCTest.t.sol 

```solidity
 function test_PoC_setUpTheDividendsRunsOutOfGasWhenFlowIsMarkedClaimed() public {
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 10;
        uint256[] memory claimedSeconds = new uint256[](1);
        claimedSeconds[0] = 1;
        uint256[] memory vestingPeriods = new uint256[](1);
        vestingPeriods[0] = 30 days;
        bool[] memory claimedFlows = new bool[](1);
        claimedFlows[0] = true;

        vesting.primeAllocation(0, IERC20(address(secondaryToken)), amounts, claimedSeconds, vestingPeriods, claimedFlows);

        bytes memory callData = abi.encodeCall(A26ZDividendDistributor.setUpTheDividends, ());
        (bool success,) = address(distributor).call{gas: 200_000}(callData);

        assertFalse(success, "setUpTheDividends should revert because the loop never increments i");
    }

```

   │   │   └─ ← [Return] Allocation({ amounts: [10], vestingPeriods: [2592000 [2.592e6]], vestingStartTimes: [], claimedSeconds: [1], claimedFlows: [true], isClaimed: false, token: 0xF62849F9A0B5Bf2913b396098F7c7019b51A820a, assignedPoolId: 0 })
    │   └─ ← [OutOfGas] EvmError: OutOfGas
    ├─ [0] VM::assertFalse(false, "setUpTheDividends should revert because the loop never increments i") [staticcall]
    │   └─ ← [Return]
    └─ ← [Return



### Mitigation

- Increment the loop counter on every iteration (e.g., switch to `for (uint256 i; i < len; ++i)` or place `++i` ahead of each `continue`).
- Add tests similar to `protocol/test/A26ZDividendDistributorPoCTest.t.sol` to cover claimed and fresh flow scenarios so regressions are caught.
  