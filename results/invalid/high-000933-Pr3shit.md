# [000933] Pool Count Off-by-One Error Makes finalizeBids() and Allocation Setting Impossible
  
  ### Summary

The condition at [line 689](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L689):

`require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());`


will cause an off-by-one error, allowing creation of the 11th pool, which will cause finalization and allocation-setting to fail completely for the entire bidding project, as the protocol will be unable to supply the required number of Merkle roots and will revert during finalization.


### Root Cause


In [vesting.sol:689](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L689), the contract uses:

`require(biddingProject.poolCount <= 10)
`

poolCount is 0-indexed, so the valid range for 10 pools is IDs 0–9.
With <= 10, the contract incorrectly allows poolCount == 10, which would create 11 pools (0–10).

Downstream logic assumes exact matching between:

* number of pools (poolCount)

* number of Merkle roots provided in finalizeBids() and updateProjectAllocations()

This extra pool makes those functions impossible to execute.

Correct condition should be:

**`require(biddingProject.poolCount < 10)`**


### Internal Pre-conditions


1. A bidding project is created and poolCount reaches 10.

2. Owner calls createPool() one more time.

3. poolCount becomes 11 due to the incorrect <= check.


### External Pre-conditions

_No response_

### Attack Path

1. Owner calls createPool() repeatedly until poolCount == 10.

2. Calls createPool() again; check incorrectly allows it.

3. poolCount becomes 11.

4. During finalizeBids():

* The contract requires merkleRoots.length == poolCount, i.e., 11 Merkle roots.

* Project owners cannot supply 11 roots if the system expects 10.

* finalizeBids() reverts, leaving the project permanently unfinalizable.

5. No allocations can be set, no NFTs can be claimed, no refunds can be claimed, and no vesting tokens can ever be distributed.


### Impact

The entire bidding project becomes permanently stuck:

* finalizeBids() always reverts

* updateProjectAllocations() always reverts

* Users cannot claim refunds

* Vesting allocations cannot be created

All users in that bidding project lose access to their allocations and refunds.



### PoC

_No response_

### Mitigation

Replace the condition at [line 689](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L689) with:

`require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project());`
  