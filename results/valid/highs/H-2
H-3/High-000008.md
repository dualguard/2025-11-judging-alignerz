# [000008] Uninitialized memory arrays in fee / split helpers revert TVS split / merge
  
  ### Summary
The memory arrays returned from [`FeesManager::calculateFeeAndNewAmountForOneTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174) and built inside [`AlignerzVesting::_computeSplitArrays`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113-L1141) are written to without first allocating them, so any write to index `i` immediately reverts with an array index out-of-bounds error, causing `AlignerzVesting::mergeTVS` and `AlignerzVesting::splitTVS` calls that rely on these helpers to revert and preventing users from splitting or merging TVSs with flows.


### Root Cause
In both `FeesManager::calculateFeeAndNewAmountForOneTVS` and `AlignerzVesting::_computeSplitArrays` the code writes to dynamic memory arrays that are never allocated, so their length stays zero while the loops assume `length == nbOfFlows > 0`.

In `FeesManager::calculateFeeAndNewAmountForOneTVS` the function returns a `uint256[] memory newAmounts` array but never allocates it with the expected length before using it:

```solidity
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {

@>  for (uint256 i; i < length; ) {
@>      newAmounts[i] = amounts[i] - feeAmount; // newAmounts is never initialized to length

    }
}
```

Similarly, in `AlignerzVesting::_computeSplitArrays` the returned `Allocation` struct contains dynamic arrays that are not allocated before being indexed:

```solidity
function _computeSplitArrays(
    Allocation storage allocation,
    uint256 percentage,
    uint256 nbOfFlows
) internal view returns (Allocation memory alloc) {
    // ...
    // alloc.amounts / alloc.vestingPeriods / ... are never initialized
@>  for (uint256 j; j < nbOfFlows; ) {
@>      alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
        alloc.vestingPeriods[j] = baseVestings[j];
        alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
        alloc.claimedSeconds[j] = baseClaimed[j];
        alloc.claimedFlows[j] = baseClaimedFlows[j];
        // ...
    }
}
```

Because neither `newAmounts` nor the `alloc.*` arrays are set to `new <type>(nbOfFlows)` / `new <type>(length)`, their lengths remain zero while the loops iterate up to `length` / `nbOfFlows`. In Solidity, accessing `array[i]` when `i >= array.length` triggers a panic and reverts the whole call, so any invocation of `FeesManager::calculateFeeAndNewAmountForOneTVS` with `length > 0` or `AlignerzVesting::_computeSplitArrays` with `nbOfFlows > 0` will revert before returning and the caller will be unable to complete the split / merge operation.


### Internal Pre-conditions
1. A TVS allocation with `amounts.length > 0` exists for the TVS being split or merged.
2. `AlignerzVesting::mergeTVS` or `AlignerzVesting::splitTVS` is called on that TVS so that it forwards the TVS’ `amounts` and `nbOfFlows` to `FeesManager::calculateFeeAndNewAmountForOneTVS` and `_computeSplitArrays`.


### External Pre-conditions
1. None.


### Attack Path
1. A user  obtains a TVS with at least one vesting flow (i.e., `amounts.length > 0`).
2. The user calls `AlignerzVesting::mergeTVS` to merge this TVS into another, or `AlignerzVesting::splitTVS` to split it into multiple TVSs.
3. Inside these functions, the protocol calls `FeesManager::calculateFeeAndNewAmountForOneTVS` (to compute fees and updated amounts) and `AlignerzVesting::_computeSplitArrays` (to construct per-TVS split allocations), both of which enter loops that attempt to write into zero-length memory arrays.
4. The first out-of-bounds write causes a revert, so the merge / split transaction fails and the TVS state is left unchanged.


### Impact
Any `AlignerzVesting::mergeTVS` or `AlignerzVesting::splitTVS` call that involves at least one flow will revert as soon as it reaches either `FeesManager::calculateFeeAndNewAmountForOneTVS` or `AlignerzVesting::_computeSplitArrays`, effectively disabling  split / merge core  functionality for TVSs. Users cannot reorganize or consolidate their vesting schedules, and fee collection for these operations is also blocked, but no direct loss of previously recorded vesting flows occurs because state updates never reach storage.


### PoC
Not provided.


### Mitigation
Initialize all dynamic memory arrays before writing into them in both helpers, and ensure per-flow fees are computed independently:

```solidity
// FeesManager::calculateFeeAndNewAmountForOneTVS
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    newAmounts = new uint256[](length);
    for (uint256 i; i < length; ++i) {
        uint256 flowFee = calculateFeeAmount(feeRate, amounts[i]);
        feeAmount += flowFee;
        newAmounts[i] = amounts[i] - flowFee;
    }
}
```

```solidity
// AlignerzVesting::_computeSplitArrays
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
    alloc.assignedPoolId = allocation.assignedPoolId;
    alloc.token = allocation.token;

@>  alloc.amounts = new uint256[](nbOfFlows);
@>  alloc.vestingPeriods = new uint256[](nbOfFlows);
@>  alloc.vestingStartTimes = new uint256[](nbOfFlows);
@>  alloc.claimedSeconds = new uint256[](nbOfFlows);
@>  alloc.claimedFlows = new bool[](nbOfFlows);

    for (uint256 j; j < nbOfFlows; ) {
        alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
        alloc.vestingPeriods[j] = baseVestings[j];
        alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
        alloc.claimedSeconds[j] = baseClaimed[j];
        alloc.claimedFlows[j] = baseClaimedFlows[j];
        unchecked { ++j; }
    }
}
```

  