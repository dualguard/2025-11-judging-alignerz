# [000432] Fully claimed positions can still be claimable for dividends due to ``rounding``.
  
  ### Summary

Fully claimed positions can still be claimable for dividends due to ``rounding``.

### Root Cause

``Dividend`` to distribute is based on ``unclaimedAmounts``:
```solidity
./A26ZDividendDistributor.sol

    function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len;) {
            (address owner, bool isOwned) = safeOwnerOf(i);
            if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts); // here
            unchecked {
                ++i;
            }
        }
        emit dividendsSet();
    }
```
``unclaimedAmountsIn`` calculated here:

```solidity
./A26ZDividendDistributor.sol

    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
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
``claimedAmount`` is rounded down:
```solidity
            uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i]; // rounded down
            uint256 unclaimedAmount = amounts[i] - claimedAmount;
            amount += unclaimedAmount;
            unchecked {
                ++i;
            }
        }
        unclaimedAmountsIn[nftId] = amount;
```
If `` claimedSeconds[i] == vestingPeriods[i]``, meaning ``full claim``, ``amounts[i]`` should be equal to ``claimedAmount``.
But due to ``rounding down``, ``amounts[i] != claimedAmount`` even if fully claimed. 

```solidity
  amount += unclaimedAmount; // some dust will be added even if fully claimed due to rounding down
```

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. User Fully Claimed their TVS Position or any one of the amount in ``amounts[]``.

### Impact

1. Fully Claimed Position will also be eligible for ``dividend`` which breaks the core ``invariant``. 

### PoC

_No response_

### Mitigation

If ``claimedSeconds`` == ``vestingPeriod``, just return the ``claimedAmount`` equal to ``original Amount``.
  