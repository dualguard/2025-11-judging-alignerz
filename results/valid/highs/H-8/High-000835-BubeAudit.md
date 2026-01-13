# [000835] Infinite loop in `A26ZDividendDistributor::getUnclaimedAmounts`
  
  ### Summary

The `A26ZDividendDistributor::getUnclaimedAmounts` function doesn't increment the loop variable `i` under certain conditions, resulting in an infinite loop and out of gas revert.

### Root Cause

The [`A26ZDividendDistributor::getUnclaimedAmounts`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161) function returns the amount of unclaimed tokens of a TVS:

```solidity

function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
    uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
    uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
    uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
    bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
    uint256 len = vesting.allocationOf(nftId).amounts.length;
    for (uint i; i < len;) {
@>      if (claimedFlows[i]) continue;
@>      if (claimedSeconds[i] == 0) {
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

The problem is that the loop only increments `i` inside the last branch, but both continue statements skip the increment. If `claimedFlows` is true or `claimedSeconds` is 0, the `i` remains the same, causing the loop to run forever and the function to revert due to out of gas.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

There is no attack path, the issue occurs every time when one of the following functions is called: `getUnclaimedAmounts`, `getTotalUnclaimedAmounts`, `setAmounts` and `setUpTheDividends`.

### Impact

The `A26ZDividendDistributor::getUnclaimedAmounts` function is used in `getTotalUnclaimedAmounts`, that is used in `_setAmounts`, that is called by the owner functions `setAmounts` and `setUpTheDividends`. This means that all of these functions are non-functional, because the `getUnclaimedAmounts` reverts due to out of gas. The users can't use the `getUnclaimedAmounts` and `getTotalUnclaimedAmounts` and the owner can't ser amounts and dividends. This is High Impact: "Severe disruption of protocol functionality or availability" and the Likelihood is High, because it will happen every time when the function is called. Therefore, the Severity is also High.

### PoC

_No response_

### Mitigation

```diff

function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
    uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
    uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
    uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
    bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
    uint256 len = vesting.allocationOf(nftId).amounts.length;
    for (uint i; i < len;) {
        if (claimedFlows[i]) {
+           ++i
            continue;
        }
        if (claimedSeconds[i] == 0) {
            amount += amounts[i];
+           ++i;
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
  