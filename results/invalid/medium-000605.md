# [000605] Merge Fees Are Incorrectly Taken From Protocol Instead of User
  
  ### Summary

The incorrect fee source in mergeTVS() will cause unauthorized loss of protocol funds for the protocol treasury, as users will trigger merges that pull fees from the protocol’s token balance instead of paying the fee themselves.


### Root Cause

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1023
Looking at the `mergeTVS` function below

```javascript
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

        uint256 nbOfNFTs = nftIds.length;
        require(nbOfNFTs > 0, Not_Enough_TVS_To_Merge());
        require(nbOfNFTs == projectIds.length, Array_Lengths_Must_Match());

        for (uint256 i; i < nbOfNFTs; i++) {
            feeAmount += _merge(mergedTVS, projectIds[i], nftIds[i], token);
        }
        token.safeTransfer(treasury, feeAmount);//@audit should we be sending from user instead?
        emit TVSsMerged(projectId, isBiddingProject, nftIds, mergedNftId, mergedTVS.amounts, mergedTVS.vestingPeriods, mergedTVS.vestingStartTimes, mergedTVS.claimedSeconds, mergedTVS.claimedFlows);
        return mergedNftId;
    }
```
`mergeTVS()` calls:

`token.safeTransfer(treasury, feeAmount);`

without specifying a from address. 

This means that all merge fees are paid using protocol funds, not user funds.


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path


1. user owns two small vesting NFTs (TVSs).

2. Calls mergeTVS().

3. Function calculates feeAmount.

4. Instead of pulling feeAmount from the user, the protocol executes:
`token.safeTransfer(treasury, feeAmount);` which transfers from the contract, not the user.

5. Treasury receives tokens, but the contract’s balance decreases.

6. user repeats the process until the protocol runs out of tokens.


### Impact

Treasury/Protocol loses tokens every time a user merges TVSs.


### PoC

_No response_

### Mitigation


Replace the incorrect transfer with a pull-from-user transfer:

`token.safeTransferFrom(msg.sender, treasury, feeAmount);`

Also `splitTVS` function has the issue

  