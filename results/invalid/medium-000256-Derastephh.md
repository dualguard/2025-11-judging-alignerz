# [000256] `AlignerzVesting::createPool` Off-By-One Error allows creation of more than intended pools
  
  ### Summary

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L689

The `AlignerzVesting::createPool` function is intended to enforce a maximum of 10 vesting pools per bidding project. However, the condition used to restrict pool creation is incorrect:

```sol
require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());
```
Because `poolCount` is incremented after pool creation, this check incorrectly allows 11 pools instead of the intended 10. This breaks a core protocol invariant and causes inconsistencies across allocation, finalization, and claiming flows.

### Root Cause

The incorrect boundary check:
`poolCount <= 10` allows `poolCount` to reach 10 and still create an additional pool. The correct strict upper bound should be:

`poolCount < 10`

This error expands the number of valid pools beyond the design limit and propagates incorrect assumptions into downstream logic that relies on `poolCount`.

Failure Path
1. Creation of 11 Pools

All 11 calls pass, demonstrating the incorrect boundary.

2. Finalization Becomes Misaligned
finalizeBids enforces:
`require(merkleRoots.length == nbOfPools, Invalid_Merkle_Roots_Length());`
If 11 pools exist, the owner must provide exactly 11 merkle roots. Providing only the intended 10 roots causes a revert, making the project impossible to finalize.

3. Claiming Uses Pool IDs Without Bounds Checks
Functions such as:

`claimNFT(uint256 projectId, uint256 poolId, ...)`
do not validate:
`require(poolId < poolCount);`
As a result:

For projects where 11 pools were created, poolId = 10 succeeds.

For projects intended to have 10 pools, users can submit poolId values such as 10 or 11, causing reads against uninitialized storage slots in vestingPools.

4. Loops in Finalization and Allocation Propagate the Error
Both:
`finalizeBids(...)`
`updateProjectAllocations(...)`
iterate from 0 to poolCount - 1. If poolCount is 11, the protocol now expects 11 allocations, 11 merkle roots, and 11 valid pools, conflicting with the intended design and documentation.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

- The project creator launches a new bidding project via `AlignerzVesting::launchBiddingProject`.
- The creator calls `AlignerzVesting::createPool` 11 times. Because the check is `poolCount <= 10` and `poolCount` is incremented after pool creation, the 11th pool creation succeeds.


### Impact

The protocol's intended invariant of 10 pools per project is broken.

Projects can reach an invalid state requiring 11 merkle roots to finalize.

Providing fewer roots results in permanent inability to close the project.

Uninitialized pool storage may be accessed during user claims.

The public interface accepts arbitrary pool IDs, and the off-by-one error widens the range of valid-but-unintended pool IDs.

Integrations and UI flows built around a fixed limit of 10 pools become unreliable.

Overall, the off-by-one error compromises correctness, consistency, and user safety across the entire bidding and vesting pipeline.

### PoC

```sol
function test_createPool_off_by_one_allows_11_pools() public {
    vm.prank(projectCreator);
    vesting.launchBiddingProject(
        address(token), address(usdt),
        block.timestamp, block.timestamp + 1_000_000,
        "0x0", true
    );

    vm.startPrank(projectCreator);
    for (uint256 i = 0; i < 11; i++) {
        vesting.createPool(0, 1_000_000 ether, 0.01 ether, false);
    }
    vm.stopPrank();

    bytes32;
    for (uint256 i = 0; i < 11; i++) {
        merkleRoots[i] = keccak256(abi.encodePacked("root", i));
    }

    vm.prank(projectCreator);
    vesting.finalizeBids(0, bytes32(0), merkleRoots, 60); // succeeds
}
```

This test demonstrates that 11 pools can be created and finalized, violating the intended maximum of 10.

### Mitigation

Replace the incorrect condition:

```sol
require(biddingProject.poolCount <= 10, Max_Pools_Reached());
```

with the correct strict bound:

```sol
require(biddingProject.poolCount < 10, Max_Pools_Reached());
```

Additionally, all pool-related external functions should include:

`require(poolId < biddingProject.poolCount, Invalid_Pool_Id());`
A constant should be introduced for clarity:

`uint256 constant MAX_POOLS = 10;`
This eliminates ambiguity, ensures consistency, and enforces the intended invariant across the codebase.
  