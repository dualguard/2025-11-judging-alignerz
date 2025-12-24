# [000358] Attacker can split into multiple 0 amount Nft to cause OOG in dividend distribution, halting the dividend distribution for affected tokens, this can occur naturally also as interaction increases
  
  ### Summary

Users are allowed to split their positions for a fee, while they will be able to mint new NFt's, which can contain 0 amounts. This will successfully inflate the number of tokens minted. It is also worth stating that tokens minted under normal operations keep increasing even when some tokens are burned. This creates an iteration issue where minting of excessive NFTs will prevent the owner from ever distributing dividends using the A26ZDividendDistributor.sol. This leads to a broken functionality.

### Root Cause

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L213-L223

```solidity
    /// @notice Internal logic that allows the owner to set the dividends for each TVS holder
    function _setDividends() internal {

@audit>>        uint256 len = nft.getTotalMinted();           // attacker trigger out of gas by minting splitting 0 out and splitting 0 further and transferring 
        for (uint i; i < len;) {

@audit>>                (address owner, bool isOwned) = safeOwnerOf(i);

@audit>>                if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
            unchecked {
                ++i;
            }
        }
        emit dividendsSet();
    }
```

Splitting can be done with 0% creating a 0 allocation amount NFT.

```solidity
 /// @notice Allows users to split a TVS
    /// @param projectId ID of the biddingProject
    /// @param percentages % in basis point of the allocated amount that will be allocated in the TVSs after the split
    /// @param splitNftId tokenId of the NFT that will be split
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
        token.safeTransfer(treasury, feeAmount);

        uint256 nbOfTVS = percentages.length;

        // new NFT IDs except the original one
        uint256[] memory newNftIds = new uint256[](nbOfTVS - 1);

        // Allocate outer arrays for the event
        Allocation[] memory allAlloc = new Allocation[](nbOfTVS);

        uint256 sumOfPercentages;
        for (uint256 i; i < nbOfTVS;) {

@audit>>                uint256 percentage = percentages[i];

@audit>>            sumOfPercentages += percentage;

@audit>>                uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);
       
          if (i != 0) newNftIds[i - 1] = nftId;
            Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
            NFTBelongsToBiddingProject[nftId] = isBiddingProject ? true : false;
            Allocation storage newAlloc = isBiddingProject ? biddingProjects[projectId].allocations[nftId] : rewardProjects[projectId].allocations[nftId];

@audit>>                _assignAllocation(newAlloc, alloc);
            allocationOf[nftId] = newAlloc;
            allAlloc[i].amounts = alloc.amounts;
            allAlloc[i].vestingPeriods = alloc.vestingPeriods;
            allAlloc[i].vestingStartTimes = alloc.vestingStartTimes;
            allAlloc[i].claimedSeconds = alloc.claimedSeconds;
            allAlloc[i].claimedFlows = alloc.claimedFlows;
            allAlloc[i].assignedPoolId = alloc.assignedPoolId;
            allAlloc[i].token = alloc.token;
            emit TVSSplit(projectId, isBiddingProject, splitNftId, nftId, allAlloc[i].amounts, allAlloc[i].vestingPeriods, allAlloc[i].vestingStartTimes, allAlloc[i].claimedSeconds);
            unchecked {
                ++i;
            }
        }
        require(sumOfPercentages == BASIS_POINT, Percentages_Do_Not_Add_Up_To_One_Hundred());
        return (splitNftId, newNftIds);
    }
```


This causes the minted amount to increase to a massive amount. With a 0 nft amount attacker can split alot more nft to create more 0 nft allocations for free ( 0 fee).


This will cause a massive iteration problem when allocating dividends. Gas is consumed for the SLOAD and SSTORE actions this compounds the iteration problem created leading to a catastrophic griefing vector.

### Internal Pre-conditions

1. Attacker splits to inflate the NFT minted. 

### External Pre-conditions

_No response_

### Attack Path

1. Attacker splits and pays a fee only for the first split; other subsequent splitting will be free
2. This increases the amount of NFT minted, triggering out of gas on a chain like Polygon (10k - 200k) is enough to achieve this.

### Impact

Broken functionality, inability to distribute dividends through the DividendDistributor.

### PoC

_No response_

### Mitigation

Enforce a minimum amount of allocation for each split NFT. Rewrite the dividend sharing function to allow partial allocations instead of using the max NFT length. 
  