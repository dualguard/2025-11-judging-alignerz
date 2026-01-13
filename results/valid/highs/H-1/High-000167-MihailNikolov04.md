# [000167] Splitting NFTs leaves a big percentage of funds in the protocol
  
  ### Summary

Lets go through the flow of [`Vesting::splitTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054) function:
1. User inputs the project Id which the NFT belongs to, The split percentages and the NFT id itself
2. Then the system gets the allocation (as storage variable) and the ERC20 token that it supports, which depends on the project type
3. Then split fee is applied 
4. After that the function starts to compute new NFTs by iterating through the split percentages but firstly it applies the percentage to the NFT that you want to split(The initial NftId that he you to split in multiple NFTs)
5. Then the new values are applied to the newly minted NFTs

The problem here arrises from the combination of 2, 4 and 5. As the initial allocation is extracted as a storage variable, it points to the corresponding storage slot(either ` biddingProjects[projectId].allocations[splitNftId]` or `rewardProjects[projectId].allocations[splitNftId]`). Afterwards when the new allocation for the initial NFT is computed it is applied to the `newAlloc` storage variable, which again points (hence reads) to the same storage slot as can be seen here (as in the first iteration `nftId` and  `splitNftId` are the same Id):
```solidity
for (uint256 i; i < nbOfTVS; ) {
            uint256 percentage = percentages[i];
            sumOfPercentages += percentage;

            uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);
            if (i != 0) newNftIds[i - 1] = nftId;
            Allocation memory alloc = _computeSplitArrays(
                allocation,
                percentage,
                nbOfFlows
            );
            NFTBelongsToBiddingProject[nftId] = isBiddingProject ? true : false;
    @>        Allocation storage newAlloc = isBiddingProject
    @>            ? biddingProjects[projectId].allocations[nftId]
    @>            : rewardProjects[projectId].allocations[nftId];
            _assignAllocation(newAlloc, alloc);
```
When the `_assignAllocation` function update the `newAlloc` variable, it will update the `allocation` variable as well (As they point to the same storage slot), leading to inevitable loss of funds for every user who splits his NFT due to the fact that his initial NFT now has way less than 100% of its initial value. Lets track the following example:
1. User wants to split his initial biding project NFT on half (Half the value remains in the initial one and half the value is translated into new one)
2. Assuming 0 fees are applied (for simplicity) the user's initial NFT now has 50% of its starting value and this stats are saved into the `biddingProjects[projectId].allocations[splitNftId]` storage slot
3. Now that the storage slot is updated the second NFT won't receive the 50% of the starting value but instead will receive a 50% of what is written in the storage slot (so 50% of 50% = 25%)
4. This results in 25% loss of the starting value of the initial NFT

### Root Cause

Root cause of this issue is the following block of code:
```solidity
 for (uint256 i; i < nbOfTVS; ) {
            uint256 percentage = percentages[i];
            sumOfPercentages += percentage;

            uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);
            if (i != 0) newNftIds[i - 1] = nftId;
            Allocation memory alloc = _computeSplitArrays(
                allocation,
                percentage,
                nbOfFlows
            );
            NFTBelongsToBiddingProject[nftId] = isBiddingProject ? true : false;
            Allocation storage newAlloc = isBiddingProject
                ? biddingProjects[projectId].allocations[nftId]
                : rewardProjects[projectId].allocations[nftId];
            _assignAllocation(newAlloc, alloc);
```
In the first iteration the `nftId == splitNftId`, which means that `biddingProjects[projectId].allocations[nftId]` and `biddingProjects[projectId].allocations[splitNftId]` are the same exact storage slot and after the `_assignAllocation` function is called, the new allocation is officially applied to the initial NFT, resulting the big loss in the following iterations

### Internal Pre-conditions

User wants to split his NFT

### External Pre-conditions

None

### Attack Path

1. User wants to split his NFT
2. A big percentage of his value is cut off because of the storage issues

### Impact

User loses most of the value of his NFT by splitting it. Also there will be a leftover funds in the protocol but this is not a big problem since it could be swept by the owner.

### PoC

-

### Mitigation

Start iterating from the back. first mint the new NFTs and apply their allocations and leave the initial NFT for last. This way there won't be any loss of value from the starting NFT and everything will go smoothly 
  