# [000491] Second Distribution of dividend will break rewards calculations in A26ZDividendDistributor contract
  
  ### Summary

When 2nd dividend is allotted to users using `_setDividends()`, it'll break the calculations if there are some unclaimed rewards already present for the user from previous allotment. 

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L218

### Root Cause

As per the below logic, `stablecoinAmountToDistribute = stablecoin.balanceOf(address(this));`. Now, if this is the second allotment, there can be some unclaimed tokens present in the contract. So, **stablecoinAmountToDistribute** calculated will contain some past rewards.

Method `_setDividends()` will later add more amount than anticipated in `dividendsOf[owner].amount`. Because **stablecoinAmountToDistribute** is higher than actual amount to be distributed. Also,  **dividendsOf[owner].amount** will be higher too because of the presence of initial unclaimed value. This will lead to more allocation of rewards than the balance of the contract. So, some users at the end won't be able to claim their dividends because of insufficient reward tokens in the `A26ZDividendDistributor` contract.

```solidity
    function _setAmounts() internal {
@>        stablecoinAmountToDistribute = stablecoin.balanceOf(address(this));
        totalUnclaimedAmounts = getTotalUnclaimedAmounts();
        emit amountsSet(stablecoinAmountToDistribute, totalUnclaimedAmounts);
    }

    function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len;) {
            (address owner, bool isOwned) = safeOwnerOf(i);
@>            if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
            unchecked {
                ++i;
            }
        }
        emit dividendsSet();
    }
```

- https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L208

- https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L218

### Internal Pre-conditions

1. The protocol is allotting 2nd dividend.
2. There should be some unclaimed rewards for at least 1 user from 1st dividend.

### External Pre-conditions

None

### Attack Path

1. Protocol allots $1000 as dividend.
2. Alice is eligible for $100 dollars. But Alice doesn’t  claim.
3. Protocol allots 2nd dividend worth $1200.
4. Now, because Alice's $100 rewards are still available in the contract, it'll assign `stablecoinAmountToDistribute` to $1300 instead of $1200.
5. Now, Dividend of each user will be calculated as per $1300. Dividend of Alice will be calculated as `dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts)`.
7. Here, already present value of $100 in `dividendsOf[owner].amount` will inflate dividend to be distributed to Alice. Which will end up with contract not having enough reward tokens to distribute to everyone.
8. Which will cause rejection of funds to people who try to claim at the end.

### Impact

There are 2 impacts:

- Some of the users won't be able to claim their rewards. Because there won't be enough funds to distribute to everyone. As, both **stablecoinAmountToDistribute** and **dividendsOf[owner].amount**  is set incorrectly.
- If Alice has claimed dividend for certain period. Then, her `claimedSeconds` won't reset to 0. She won't be able to fully claim her 2nd Dividend allocation.

### PoC

_No response_

### Mitigation

We can keep a mapping of `DividendId`. So, `startTime`, `totalUnclaimedAmounts`, `stablecoinAmountToDistribute`, `dividendsOf` & `unclaimedAmountsIn` can be unique for each `DividendId`
  