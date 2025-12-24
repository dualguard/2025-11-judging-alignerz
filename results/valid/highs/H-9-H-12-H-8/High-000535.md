# [000535] getUnclaimedAmounts() incorrect implementation
  
  ## Summary

The function `getUnclaimedAmounts()` has some errors that prevent it from correctly implementing its functionality.



## Root cause

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
1@=>        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
        uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
        uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
        uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
        bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
        uint256 len = vesting.allocationOf(nftId).amounts.length;
        for (uint i; i < len;) {
2@=>            if (claimedFlows[i]) continue;
            if (claimedSeconds[i] == 0) {
                amount += amounts[i];
                continue;
            }
            uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
            uint256 unclaimedAmount = amounts[i] - claimedAmount;
            amount += unclaimedAmount;
            unchecked {
                ++i;
            }
        }
        unclaimedAmountsIn[nftId] = amount;
    }
```



First, `vesting.allocationOf(nftId)` will revert because the automatically generated getter for the `allocationOf` mapping incorrectly encodes the structure when called from another contract.

 What the trace shows:

```bash
 1 [3604] ERC1967Proxy::fallback(1) [staticall]
   2  ├─ [2867] AlignerzVesting::allocationOf(1) [delegatecall]
   3  │  └─ ← [Return] false, Aligners26: [0x2e...], 0
   4  └─ ← [Return] false, Aligners26: [0x2e...], 0
```

Only 3 values are returned (isClaimed, token, assignedPoolId), and 5 dynamic arrays are missing (amounts, vestingPeriods, vestingStartTimes, claimedSeconds, claimedFlows).



Next, at (1) the condition logically incorrect. The function should return zero if the tokens don't match, not the other way around.

At(2) the operator `continue` executes without incrementing `i`. This will lead to an infinite loop.



## Impact

Incorrect implementation of the function leads to DoS.



## Mitigation

```solidity
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        IAlignerzVesting.Allocation memory allocation = vesting.getAllocation(nftId);
        if (address(token) != address(allocation.token)) return 0;
        uint256[] memory amounts = allocation.amounts;
        uint256[] memory claimedSeconds = allocation.claimedSeconds;
        uint256[] memory vestingPeriods = allocation.vestingPeriods;
        bool[] memory claimedFlows = allocation.claimedFlows;
        uint256 len = allocation.amounts.length;
        for (uint i; i < len;) {
            if (claimedFlows[i]) {
                unchecked { ++i; }
                continue;
            }
            if (claimedSeconds[i] == 0) {
                amount += amounts[i];
                unchecked { ++i; }
                continue;
            }
            uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
            uint256 unclaimedAmount = amounts[i] - claimedAmount;
            amount += unclaimedAmount;
            unchecked {
                ++i;
            }
        }
        unclaimedAmountsIn[nftId] = amount;
    }
```

Add the`getAllocation` function to `src/contracts/vesting/AlignerzVesting.sol`

```solidity
function getAllocation(uint256 nftId) external view returns (Allocation memory allocation) {
        Allocation storage alloc = allocationOf[nftId];

        // Explicitly construct memory struct to ensure all fields are copied
        allocation = Allocation({
            amounts: alloc.amounts,
            vestingPeriods: alloc.vestingPeriods,
            vestingStartTimes: alloc.vestingStartTimes,
            claimedSeconds: alloc.claimedSeconds,
            claimedFlows: alloc.claimedFlows,
            isClaimed: alloc.isClaimed,
            token: alloc.token,
            assignedPoolId: alloc.assignedPoolId
        });
    }

}
```

  