# [000308] A26ZDividendDistributor contract can be DoS intentionally or unintentionally, lead to rewards cannot be allocated to TVS holders 
  
  ### Summary

As we can see how `A26ZDividendDistributor` contract work, first owner set up the reward by transfer the reward token to the contract. Then owner call `setUpTheDividends()` function, there are 2 internal function get called after that. First one, `_setAmounts()` and the second is `_setDividends()` . `_setAmounts()` can be seen below :

```solidity
function _setAmounts() internal {
        stablecoinAmountToDistribute = stablecoin.balanceOf(address(this));
        totalUnclaimedAmounts = getTotalUnclaimedAmounts();
        emit amountsSet(stablecoinAmountToDistribute, totalUnclaimedAmounts);
    }
```

If we see on `getTotalUnclaimedAmounts()` function, len represents all NFTs that have been minted. This function will iterate as many times as the total number of NFTs that have been minted.

```solidity
function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) { 
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len;) { 
            (, bool isOwned) = safeOwnerOf(i);
            if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
            unchecked {
                ++i;
            }
        }
    }
```

This will cause problems if many NFTs that have been minted and ends up with iterations that cause OOG. This is known as unbounded loop.

Note

This issue affect `setDividends()` function too, because in the `_setDividends()` it loop for all of NFTs minted too

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
And note that, `getTotalMinted()` include all minted NFT even the NFT already burned, this can be seen below :

```solidity
function getTotalMinted() external view returns (uint256 totalMinted){
        totalMinted = _totalMinted();
    }

function _totalMinted() internal view returns (uint256) {
        // Counter underflow is impossible as _currentIndex does not decrement,
        // and it is initialized to _startTokenId()
        unchecked {
            return _currentIndex - _startTokenId();
        }
    }
```

### Root Cause

In [A26ZDividendDistributor.sol:128](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L128) there is unbounded loop that may lead to OOG revert

### Internal Pre-conditions

-

### External Pre-conditions

-

### Attack Path

This issue can be triggered intentionally or unintentionally. 

A. Intentionally

1. Attacker is a TVS holder, he can split the TVS he owns with the `splitTVS()` function on the `AlignerzVesting.sol` contract
2. This way the total of NFTs minted will greatly increase
3. Lead to `getTotalUnclaimedAmounts()` revert because of OOG 

B. Unintentionally

For this, it could happen because of the large number of NFTs that have been minted (more project -> more winner -> more NFT minted) and causes the `getTotalUnclaimedAmounts()` function revert because of OOG

### Impact

The main function for `A26ZDividendDistributor.sol` contract can be DoS intentionally or unintentionally and lead to rewards cannot be allocated to TVS holders.


### PoC

_No response_

### Mitigation

_No response_
  