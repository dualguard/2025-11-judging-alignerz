# [000435] ``calculateFeeAndNewAmountForOneTVS()`` doesn't work as intended and leads to loss of TVS amount.
  
  ### Summary

``calculateFeeAndNewAmountForOneTVS()`` doesn't work as intended and leads to loss of TVS amount.

### Root Cause

``calculateFeeAndNewAmountForOneTVS()`` is implemented as:

```solidity
./FeesManager.sol:

    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }


    function calculateFeeAmount(uint256 feeRate, uint256 amount) public pure returns(uint256 feeAmount) {
        feeAmount = amount * feeRate / BASIS_POINT;
    }
```
Notice that ``i++`` is missing. Meaning the loop will only be executed once. But it takes ``amounts[]`` as input. Also, the ``newAmounts[]`` it returns will only have ``the first amount``.

This function is used in:
```solidity
./AlignerzVesting.sol

    function splitTVS(
        uint256 projectId,
        uint256[] calldata percentages,
        uint256 splitNftId
    ) external returns (uint256, uint256[] memory) {
        address nftOwner = nftContract.extOwnerOf(splitNftId);
        require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());

        bool isBiddingProject = NFTBelongsToBiddingProject[splitNftId];
        (Allocation storage allocation, IERC20 token) = isBiddingProject ?
        (biddingProjects[projectId].allocations[splitNftId], biddingProjects[projectId].token) :
        (rewardProjects[projectId].allocations[splitNftId], rewardProjects[projectId].token);

        uint256[] memory amounts = allocation.amounts;
        uint256 nbOfFlows = allocation.amounts.length;
        (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);
        allocation.amounts = newAmounts;
...
```
and 

```solidity
./AlignerzVesting.sol

    function mergeTVS(uint256 projectId, uint256 mergedNftId, uint256[] calldata projectIds, uint256[] calldata nftIds) external returns(uint256) {
        address nftOwner = nftContract.extOwnerOf(mergedNftId);
        require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());
        
        bool isBiddingProject = NFTBelongsToBiddingProject[mergedNftId];
        (Allocation storage mergedTVS, IERC20 token) = isBiddingProject ?
        (biddingProjects[projectId].allocations[mergedNftId], biddingProjects[projectId].token) :
        (rewardProjects[projectId].allocations[mergedNftId], rewardProjects[projectId].token);

        uint256[] memory amounts = mergedTVS.amounts;
        uint256 nbOfFlows = mergedTVS.amounts.length;
        (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
        mergedTVS.amounts = newAmounts;
...
```
We can see that the ``newAmounts[]`` returned by  ``calculateFeeAndNewAmountForOneTVS`` is directly set to ``storage``. 

This is very problematic and overrides multiple ``amounts`` into ``one``.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. ``TVS`` owners need to have ``merged`` two ``nfts`` into ``one`` which will have ``amounts`` from ``nftId1`` and ``nftId2`` using the ``merge()`` function.

> Step 1 will work correctly as at first there is only one ``nftId``.

2. Now if they want to ``merge`` or ``split`` their ``nftId`` with ``multiple amounts`` again, only the first amount will ``remain``.

> Note: There is no ``limit`` to the amount of ``nftIds`` that can be ``merged``.

### Impact

1. ``nftId1`` has ``allocations[nftId1].amounts[0] = 10``.
2. ``nftId2`` has ``allocations[nftId2].amounts[0] = 20``.
3.  Both are merged into ``nftId1`` and the allocations will be:
a.  ``allocations[nftId1].amounts[0] = 10`` 
b. ``allocations[nftId1].amounts[1] = 20``
4. Now, user wants to merge ``nftId3`` with ``allocations[nftId3].amounts[0] = 30`` to ``nftId1``.
5. ``calculateFeeAndNewAmountForOneTVS()`` for ``nftId1`` will return ``newAmounts[0] = 10`` only. 
6. ``allocations[nftId1].amounts[1] = 20`` is lost forever.

### PoC

_No response_

### Mitigation

Iterate over all the ``amounts[]``, not only the first one.
  