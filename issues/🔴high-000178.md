# [000178] Impossibility to set amounts and dividends in A26ZDividendDistributor contract due to out of gas error.
  
  ### Summary

`setUpTheDividends`, `setAmounts` and `setDividends` functions are 3 `onlyOwner` functions, used to setup a round of dividends distribution. The first one is a combination of the 2 others.

```solidity
    function setUpTheDividends() external onlyOwner {
        _setAmounts();
        _setDividends();
    }
```

`_setAmounts` and `_setDividends` are defined as follows:

```solidity
    function _setAmounts() internal {
        stablecoinAmountToDistribute = stablecoin.balanceOf(address(this));
        // @audit HIGH REPORTED out of gas
        totalUnclaimedAmounts = getTotalUnclaimedAmounts();
        emit amountsSet(stablecoinAmountToDistribute, totalUnclaimedAmounts);
    }

    function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
        uint256 len = nft.getTotalMinted();
        for (uint256 i; i < len;) {
            (, bool isOwned) = safeOwnerOf(i);
            if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
            unchecked {
                ++i;
            }
        }
    }
```

```solidity
    function _setDividends() internal {
        // @audit HIGH REPORTED out of gas
        uint256 len = nft.getTotalMinted();
        for (uint256 i; i < len;) {
            (address owner, bool isOwned) = safeOwnerOf(i);
            if (isOwned) {
                dividendsOf[owner].amount += (unclaimedAmountsIn[i]
                        * stablecoinAmountToDistribute
                        / totalUnclaimedAmounts);
            }
            unchecked {
                ++i;
            }
        }
        emit dividendsSet();
    }
```

Both of them iterate over `nft.getTotalMinted()` which is a very bad idea and will inevitably lead to a complete DOS of the contract.

### Root Cause

`_setAmounts`  especially, which does a lot of computation (calls `getTotalUnclaimedAmounts` which calls `safeOwnerOf` that does an external call, and `getUnclaimedAmounts`) is the root cause. Calling it will very quickly exceed the block gas limit of 45 millions gas.

### Attack Path

The more project launch on the platform, or the more split happen, the more NFTs are minted. Once more than a few hundrands NFTs are minted, the contract will be unusable.

### Impact

The impact is high as it results in a complete DOS of the functionalities of the contract once enough NFTs have been minted.

### Mitigation

Rework the way you compute unclaimed amounts related to a given token.
  