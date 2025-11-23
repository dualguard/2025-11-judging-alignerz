# [000177] Incorrect check in `getUnclaimedAmounts` function in A26ZDividendDistributor contract leads to rewards distributed to wrong users.
  
  ### Summary

The purpose of the A26ZDividendDistributor contract is to distribute dividends to users that own a TVS that vests a specific token and with unclaimed amounts. It will be mainly use to distribute dividends to holders of TVS for the A26Z token. But the developer said it can be changed to reward TVSs for another token.

The `getUnclaimedAmounts` function is defined as follows:

```solidity
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        // @audit HIGH REPORTED: should be !=, leads to rewards distributed to wrong users
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
        uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
        uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
        uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
        bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
        uint256 len = vesting.allocationOf(nftId).amounts.length;
        for (uint256 i; i < len;) {
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

This function is called when the owner sets the amounts with `setAmounts` or `setUpTheDividends` for the current round of dividend distribution.

The issue is that the line:
```solidity
if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
```
is incorrect, and do the opposite as expected.


### Root Cause

The root cause is the incorrect check in `getUnclaimedAmounts` function. When setting amounts and iterating over all minted NFTs, the current implementation will only count NFTs not related to the token instead of NFTs of the token. The line:
```solidity
unclaimedAmountsIn[nftId] = amount;
```
will attribute rewards only to NFTs that are not related to the token for which there is a dividend distribution.

### Attack Path

There is no specific attack path. Logic is broken for rewards repartition, currently giving rewards to all NFTs not related to the token for which there is dividends.

### Impact

The impact of this issue is high.


### Mitigation

Modify the `getUnclaimedAmounts` function check so that it correctly returns 0 if addresses are different:

```solidity
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
 >     if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;
        uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
        uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
        uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
        bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
        uint256 len = vesting.allocationOf(nftId).amounts.length;
        for (uint256 i; i < len;) {
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
  