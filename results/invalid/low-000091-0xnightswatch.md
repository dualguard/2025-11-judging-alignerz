# [000091] Confusing event emissions when finalizeBids is called with zero merkle roots
  
  ### Summary

The emission of `PoolAllocationSet` events with bytes32(0) values in `finalizeBids` will cause confusion for off-chain systems as the owner will call `finalizeBids()` with zero merkle roots to close bidding first, then call `updateProjectAllocations()` to set actual roots later, resulting in duplicate event emissions for the same pools.

### Root Cause

In `AlignerzVesting.sol:795-798`, the `finalizeBids()` emits `PoolAllocationSet` events for each pool regardless of whether the merkle root is bytes32(0) or an actual value. When the owner follows the intended workflow of calling `finalizeBids()` with zero roots first, then `updateProjectAllocations()` later, each pool will emit PoolAllocationSet twice - once with bytes32(0) and once with the actual root.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L822-L824


### Internal Pre-conditions

1. Owner needs to launch a bidding project and create pools
2. Owner needs to follow the workflow of calling `finalizeBids()` with bytes32(0) merkle roots to close bidding
3. Owner needs to later call `updateProjectAllocations()` with actual merkle roots

### External Pre-conditions

N/A

### Attack Path

1. Owner calls finalizeBids(projectId, refundRoot, [bytes32(0), bytes32(0)], claimWindow) to close bidding
     - Loop iterates through pools and emits `PoolAllocationSet(projectId, 0, bytes32(0))` and `PoolAllocationSet(projectId, 1, bytes32(0)) `
     - Project is marked as closed

2.  Owner calls `updateProjectAllocations(projectId, refundRoot, [actualRoot1, actualRoot2])` to set real merkle roots
      - Loop iterates through pools again and emits `PoolAllocationSet(projectId, 0, actualRoot1)` and `PoolAllocationSet(projectId, 1, actualRoot2)` 
 
3. Off-chain systems listening to `PoolAllocationSet` events receive duplicate events for each pool, first with zero roots then with actual roots, causing confusion about which merkle root is valid

### Impact

* Duplicate `PoolAllocationSet` events are emitted per pool, creating ambiguity for off-chain systems.
* The first emission contains `bytes32(0)` fields, which can be misinterpreted as “no allocations.”
* Integrators must add custom logic to filter out the empty events and identify the valid merkle root, increasing operational complexity.


### PoC

Not applicable - the issue is straightforward from code inspection and the owner's stated workflow. (mentioned on discord)

### Mitigation

Only emit PoolAllocationSet when setting non-zero merkle roots

Also, if the intended behavior as per discord that the intended behavior is to set it to zero then the value of merkleRoot are by default bytes32(0). 
  