# [000492] Method A256ZDividendDistributor::getUnclaimedAmounts() applies token check incorrectly
  
  ### Summary

Method `A256ZDividendDistributor::getUnclaimedAmounts()` is used to calculated the amount of certain tokens hold by each NftId. But it returns 0 instead.
```solidity
if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
```

### Root Cause

The below method returns 0 when `address(token) == address(vesting.allocationOf(nftId).token)`

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

The method will calculate `unclaimedAmountsIn` mapping incorrectly every time `A256ZDividendDistributor::getUnclaimedAmounts()` is called.

### Impact

The method `A256ZDividendDistributor::getUnclaimedAmounts()` skips target token holders. Instead it calculates `unclaimedAmountsIn` for other token holders. These tokens can have different decimals. Hence tokens will be distributed disproportionately to every other token holder. 

### PoC

_No response_

### Mitigation

```diff
   /// @notice USD value in 1e18 of all the unclaimed tokens of a TVS
    /// @param nftId NFT Id
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
--      if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
++      if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;
        uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
        uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
        uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
        bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
        uint256 len = vesting.allocationOf(nftId).amounts.length;
        for (uint i; i < len;) {
            if (claimedFlows[i]) continue;
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
  