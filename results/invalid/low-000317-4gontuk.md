# [000317] Unused extra refund flag in bidding pools leads to misleading configuration
  
  ### Description:
The `AlignerzVesting` bidding pool struct includes a `bool hasExtraRefund` flag that is set on pool creation and emitted in `PoolCreated()`, but it is never read anywhere in the contract logic. As a result, configuring this flag or monitoring it off‑chain has no effect on how refunds actually work.

For example, in [`createPool()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L679-L702) the flag is stored and emitted:

```solidity
struct VestingPool {
    bytes32 merkleRoot;
    bool hasExtraRefund;
}

function createPool(
    uint256 projectId,
    uint256 totalAllocation,
    uint256 tokenPrice,
    bool hasExtraRefund
) external onlyOwner {
    ...
    uint256 poolId = biddingProject.poolCount;
    biddingProject.vestingPools[poolId] = VestingPool({
        merkleRoot: bytes32(0),
        hasExtraRefund: hasExtraRefund
    });
    biddingProject.poolCount++;
    emit PoolCreated(projectId, poolId, totalAllocation, tokenPrice, hasExtraRefund);
}
```

However, the refund implementation `claimRefund()` never consults `hasExtraRefund`; it uses a global `refundRoot` and a hardcoded `poolId` instead, so all pools behave the same regardless of the flag’s value. This creates a misleading configuration surface where integrators may assume that turning `hasExtraRefund` on or off changes the refund behavior for that pool, when in fact it does nothing.

### Impact:
The `hasExtraRefund` configuration and event parameter are misleading and unused, which can cause incorrect assumptions in off-chain tooling or operator workflows.

### Recommendation:
Either remove the `hasExtraRefund` field, parameter, and event argument entirely to avoid confusion, or implement the intended extra-refund behavior by branching refund logic on `vestingPools[poolId].hasExtraRefund` (e.g., using distinct Merkle roots or amounts for pools that should also refund winners).

  