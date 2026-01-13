# [000834] More dividends to the TVS holders can be set than intended
  
  ### Summary

The owner can set more dividends to the TVS holders than intended.

### Root Cause

The owner sets the dividends that should be distributed to the TVS holders through the functions: [`A26ZDividendDistributor::setDividends`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L122C14-L122C26) and [`A26ZDividendDistributor::setUpTheDividends`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L111). The both functions have `onlyOwner` modifier and call the internal function [`_setDividends`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L214-L223) that have the following implementation:

```solidity

function _setDividends() internal {
    uint256 len = nft.getTotalMinted();
    for (uint i; i < len;) {
        (address owner, bool isOwned) = safeOwnerOf(i);
        if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
        unchecked {
            ++i;
        }
    }
    emit dividendsSet();
}

```

The problem here is that the function flow doesn't check if the dividends are already set. The owner is trusted and the owner will not act in a malicious way, but it is possible the owner to call by mistake the `setDividends` or `setUpDividends` function again. In that way the `amount` is not overwritten, but it adds the new value to the previously set one. This means the users can claim more dividends than the intended. Additionally, when the users start to claim their dividends, the funds will be not sufficient for all of them due to the incorrect assigned amount values and the [`claimDividends`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L202) function reverts due to the insufficient balance. 

There is one more thing about setting the dividens, but the probability to happen is very very low, because it is owner action, but it is worth to mention. If the owner forget to set dividends and the user tries to claim dividends during the vesting period, the `claimedSeconds` variable will be set to the passt seconds, but the user will receive 0 tokens for this period. This leads to loss of funds for the users.


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

It is not a malicious attack path. The owner can by mistake set again the dividends and in that way the allocated dividends will be more than the intended and only the first users will be able to claim them.

### Impact

The `claimDividends` function reverts due to insufficient balance and some users receive more funds than intended, in the same time other users are unable to receive their dividends. Lost funds for the protocol and users. The likelihood of this issue is Low, because it is owner action, but if it happens only once, the Immpact is very High for the protocol and user, therefore I think Medium severity is appropriate here.

### PoC

_No response_

### Mitigation

Add a variable that shows if the dividends are set for the required time.
  