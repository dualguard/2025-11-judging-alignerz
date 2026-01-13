# [000487] Method AlignerzVesting::splitTVS() applies fees incorrectly
  
  ### Summary

When a user calls `AlignerzVesting::splitTVS()`, `splitFeeRate` is applied. But it is applied to the `allocation.amounts`. But part of this amounts might be claimed by users already. If so, the protocol will take out more amount as fee from the `AlignerzVesting.sol` contract causing a deficit of tokens.

```solidity
        (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);
        token.safeTransfer(treasury, feeAmount);
```

### Root Cause

The protocol split fee is calculated incorrectly. Which will transfer more fees to treasury and will be charged less to the owner of the NFT.

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1069-L1071

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

A user can claim part of their vesting tokens and then later use `AlignerzVesting::splitTVS()`. 
For example:

1. Alice has $100 in vesting and She already claimed 50% of it. 
2. The splitFee is 10%.
3. Later Alice decides to split its NFT. The Split fee is charged on $100. Which will be $10. The amount left is $90 now.
4. Now, when Alice claims rest of the 50%. Which will come out at $45.
5. So, in total Alice claimed $95 and $10 was paid to treasury. 
6. So, in total of $105 are used. Even though Alice allocation was $100. 
7. This creates a deficit of $5 in the `AlignerzVesting.sol` contract.

### Impact

Some users won't be able to claim their tokens because of deficit of tokens in the `AlignerzVesting.sol` contract.

### PoC

_No response_

### Mitigation

Only charge fees on the unclaimed amounts in biddingProjects[projectId].allocations[nftId] mapping.
  