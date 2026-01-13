# [000236] Token Price Parameter Accepted and Validated But Never Stored in Pool Creation
  
  - **Description**

The `AlignerzVesting::createPool()` function accepts a `tokenPrice` parameter in its function signature and validates that it's greater than zero, but never stores this value anywhere in the contract. The `VestingPool` struct only contains `merkleRoot` and `hasExtraRefund` fields - there is no field for storing the token price.

While the price is emitted in the `PoolCreated` event, the contract has no on-chain record of it after the transaction completes. This creates a situation where off-chain systems reading events believe there's a price enforcement mechanism, but the contract cannot actually enforce or reference this price in any way.

- **Impact**

- Off-chain systems and UIs may display token prices based on events, but these prices have no on-chain enforcement
- Users may trust the emitted price when making bidding decisions, but actual allocations (determined by merkle trees) can use completely different prices
- Wasted gas validating and emitting a parameter that serves no purpose
- Suggests incomplete feature implementation that may confuse developers and auditors
- Gap between what the event suggests (price enforcement) and what the contract actually does (no price tracking)

- **Recommended Mitigation**

**Option 1:** If token pricing is not needed, remove the parameter entirely:

```solidity
function createPool(uint256 projectId, uint256 totalAllocation, bool hasExtraRefund)
    external
    onlyOwner
{
    require(projectId < biddingProjectCount, Invalid_Project_Id());
    require(totalAllocation > 0, Zero_Value());
    // Remove tokenPrice validation

    BiddingProject storage biddingProject = biddingProjects[projectId];
    require(!biddingProject.closed, Project_Already_Closed());
    require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());

    biddingProject.token.safeTransferFrom(msg.sender, address(this), totalAllocation);

    uint256 poolId = biddingProject.poolCount;
    biddingProject.vestingPools[poolId] = VestingPool({
        merkleRoot: bytes32(0),
        hasExtraRefund: hasExtraRefund
    });

    biddingProject.poolCount++;
    emit PoolCreated(projectId, poolId, totalAllocation, hasExtraRefund);
}
```

**Option 2:** If pricing should be tracked, update the VestingPool struct:

```solidity
struct VestingPool {
    bytes32 merkleRoot;
    bool hasExtraRefund;
    uint256 tokenPrice;
}
```
  