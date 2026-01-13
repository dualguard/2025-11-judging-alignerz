# [001044] More pools can be created instead of intended amount
  
  ### Summary

Current `AlignerzVesting::createPool` allows 11 distinct pools to be created instead of the intented amount (10)

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L679-L702

### Root Cause

Current check: require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());
```solidity
require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());
```
Then the code uses poolId = biddingProject.poolCount; ...; biddingProject.poolCount++;
```solidity
        require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());

        biddingProject.token.safeTransferFrom(msg.sender, address(this), totalAllocation);

        uint256 poolId = biddingProject.poolCount;
        biddingProject.vestingPools[poolId] = VestingPool({
            merkleRoot: bytes32(0), // Initialize with empty merkle root
            hasExtraRefund: hasExtraRefund
        });
        //@audit allows 11 pools due to <= above
        biddingProject.poolCount++;
```
Example:
Start poolCount = 0 → check 0 <= 10 passes → create poolId 0 → increment → poolCount = 1
...
When poolCount = 10 → check 10 <= 10 passes → create poolId 10 → increment → poolCount = 11
Result: indices 0..10 are usable → 11 distinct pools were created. If the intended maximum is 10 pools,

### Internal Pre-conditions

N/A

### External Pre-conditions

N/A

### Attack Path

N/A

### Impact

Allows one extra pool beyond the intended cap

### PoC

N/A

### Mitigation

Replace the <= check with a strict < check so the code only allows poolCount values 0..9 (10 pools total):

require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project());
  