# [000165] User can game the reward accrual to receive more funds and block other users from claiming dividends
  
  ### Summary

This is possible due to the way rewards are accrued. Lets track the following example to clear the picture:

1. Owner calls the [`A26ZDividendDistributor::setUpTheDividends()`](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L111C14-L111C31) function to set up the dividends
2. Lets assume that there are 10 minted NFTs (10 people holding 10 NFTs which all have 10% share of the `totalUnclaimedAmounts` for simplicity)
3. Now that the dividends are distributed and everybody holds 10% of the `totalUnclaimedAmounts` this means that every NFT holder must claim 10% of the stablecoin balance of the contract (Assuming it is 1e7, again for simplicity).
4. 9 out of 10 people claim their share and now there is 1e6 amount if stablecoin in the contract
5. The malicious user waits until there is a second distribution period 
6. Again in the second period there are the same 10 people with 10% share of the `totalUnclaimedAmounts` this time the contract balance of the stablecoin + the leftover funds from the malicious user (Assuming 1e6 + 1e7 as this is the amount the owner will add)
7. Now everybody must receive 1,1e6 number of tokens as dividend for this round but the malicious user now has 2,1e6 (last round + current round dividends)
8. Now when he claims, the contract will remain with 8,9e6 balance of stablecoin, leading to DoS for the last claiming user to claim

This scenario will occur in any conditions not just in the one described. Even if the owner doesn't provide any additional funds for the second dividend period, the malicious user will end up with inflated dividends and will perform the attack in some of the following rounds of the distribution. It can be backed with the way `_setDividends` function is operating, as it is adding the newly accrued amounts into the `dividendsOf[owner].amount` mapping.

### Root Cause

Root cause of the issue us the use of `stablecoin.balanceOf(address(this))` in the `_setAmounts` and the fact that the newly accrued dividends are added to the already accrued ones as can be seen here:
```solidity
    function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len; ) {
            (address owner, bool isOwned) = safeOwnerOf(i);
            if (isOwned)
                //FIXME: This leads to more claims for users that claim later
@>                dividendsOf[owner].amount += ((unclaimedAmountsIn[i] *  <@
                    stablecoinAmountToDistribute) / totalUnclaimedAmounts);
            unchecked {
                ++i;
            }
        }
        emit dividendsSet();
    }
```

### Internal Pre-conditions

User has a share of the `totalUnclaimedAmounts` in distribution number `N` and waits to claim at least until distribution `N + 1` (The actual number of distributions isn't counted so this can be translated as "when the owner calls the `setUpTheDividends()` function")

### External Pre-conditions

None

### Attack Path

1. User has a share of the `totalUnclaimedAmounts` in distribution number `N`
2. He doesn't claim until the next distribution 
3.  He claims in the next or one of the following distributions

### Impact

There are 2 HIGH impacts:
1. User ends up with more tokens than he should (in the provided example he ends up with 2,1e6 when he must end up with 2e6)
2. User is able to brick the claim function for the user who claims last

### PoC

None

### Mitigation

either forcefully send the dividends to the users after the end of the period or doesn't add the newly accrued dividends to the to the already accrued ones in the `_setDividends` function, like this:
```diff
 function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len; ) {
            (address owner, bool isOwned) = safeOwnerOf(i);
            if (isOwned)
                //FIXME: This leads to more claims for users that claim later
-                dividendsOf[owner].amount += ((unclaimedAmountsIn[i] *
+                dividendsOf[owner].amount = ((unclaimedAmountsIn[i] *
                    stablecoinAmountToDistribute) / totalUnclaimedAmounts);
            unchecked {
                ++i;
            }
        }
        emit dividendsSet();
    }
```
This way the user will lose his dividends from the first round but it is his fault that he didn't claim the anyways.
  