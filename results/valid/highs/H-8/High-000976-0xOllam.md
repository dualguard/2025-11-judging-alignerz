# [000976] 🔴 High getUnclaimedAmounts() has an infinite loop if claimedFlows is true or if claimedSeconds is 0
  
  ### Summary

getUnclaimedAmounts() does not properly increment i when either of it's "continue" branches are reached, locking it into an infinite loop. This makes _setAmounts() entirely useless if anyone has claimed, which makes it impossible to call setDividends(), bricking core protocol functionality.

### Root Cause

From A26ZDividentDistributor.sol line 140:

Please note how in both of the branches that end with "continue" i is not incremented. The transaction will loop forever using that i until it runs out of gas.

```solidity
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
        uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
        uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
        uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
        bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
        uint256 len = vesting.allocationOf(nftId).amounts.length;
        for (uint i; i < len;) {
            if (claimedFlows[i]) continue; // @audit-issue infinite loop
            if (claimedSeconds[i] == 0) {
                amount += amounts[i]; // @audit-issue infinite loop
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

isClaimed is set to true if a user has fully claimed, and claimedSeconds is 0 by default. The only way this doesn't revert is if every single owner of an nft has partially claimed.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

NA

### Impact

Admin cannot pay out dividends.

### PoC

_No response_

### Mitigation

Increment i before each continue
  