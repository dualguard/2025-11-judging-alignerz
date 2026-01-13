# [000029] `mergeTVS` leaves `allocationOf` stale, causing incorrect dividend distribution
  
  ### Summary

`mergeTVS` updates the merged TVS allocation only in the internal project storage (`biddingProjects[projectId].allocations` / `rewardProjects[projectId].allocations`) but does **not** update the public `allocationOf` mapping. External contracts such as `A26ZDividendDistributor` rely on `allocationOf` to compute unclaimed TVS balances per NFT, which are then used to allocate stablecoin dividends. 

```solidity
function getUnclaimedAmounts(
    uint256 nftId
) public returns (uint256 amount) {
    if (address(token) == address(vesting.allocationOf(nftId).token))
        return 0;
    uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
    // ... snip ...
    unclaimedAmountsIn[nftId] = amount;
}
```

After a merge, this creates a divergence between the *real* vesting state and the state visible through `allocationOf`, leading to systematically incorrect dividend calculations for TVS holders.

### Root Cause

Not updating `allocationOf` for the merged NFT when merging TVS, https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

N/A

### Impact

The holder of the merged NFT would receive an incorrect dividend amount.

### PoC

_No response_

### Mitigation

Consider updating the `allocationOf` to be in sync with the updated `mergedTVS` allocation.
  