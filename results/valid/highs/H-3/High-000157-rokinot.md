# [000157] Merge and split function does not work due to lack of dynamic array initialization
  
  _computeSplitArrays does not work as intended. It declares a memory struct alloc to return, but it never initializes the dynamic arrays within that struct.

When _computeSplitArrays is called, Solidity creates an Allocation struct named alloc in memory.
For memory structs, any dynamic arrays inside them (like amounts, vestingPeriods, claimedSeconds, etc.) are created with a length of zero. The function then enters a for loop and immediately tries to write to alloc.amounts[j]. Since j is 0 and the alloc.amounts array has a length of 0, this is an out-of-bounds access, which causes the panic (0x32) error.

```solidity
    function _computeSplitArrays(
        Allocation storage allocation,
        uint256 percentage,
        uint256 nbOfFlows
    )
        internal
        view
        returns (
            Allocation memory alloc
        )
    {
        uint256[] memory baseAmounts = allocation.amounts;
        uint256[] memory baseVestings = allocation.vestingPeriods;
        uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
        uint256[] memory baseClaimed = allocation.claimedSeconds;
        bool[] memory baseClaimedFlows = allocation.claimedFlows;
        alloc.assignedPoolId = allocation.assignedPoolId;
        alloc.token = allocation.token;
        for (uint256 j; j < nbOfFlows;) {
            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT; //<--- reverts here
```

To fix this, you must explicitly create the arrays in memory with the correct size (nbOfFlows) before the loop begins. The same bug happens in the mergeTVS function
  