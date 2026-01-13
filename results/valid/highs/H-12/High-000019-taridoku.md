# [000019] High: Inverted token filter in `A26ZDividendDistributor.getUnclaimedAmounts`
  
  ### Summary

The function attempts to only count unclaimed amounts for NFTs whose `allocation.token` matches the dividend token. Instead, the logic is flipped, thereby checking for unclaimed amounts of tokens != dividend token

### Root Cause

`A26ZDividendDistributor` is initialized with ` address _stablecoin` and  `address _token` in its [constructor](https://github.com/dualguard/2025-11-alignerz-taridoku/blob/bca8c6f4be40b274a1c40cd2016a545e622b045c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L76-L77), where: 
- stablecoin = asset being distributed as dividends
- token = the TVS token underlying the vesting positions this distributor is responsible for
In `AlignerzVesting`, whenever an NFT is minted for a vesting position, `Allocation.token` field is set, as seen in [reward](https://github.com/dualguard/2025-11-alignerz-taridoku/blob/bca8c6f4be40b274a1c40cd2016a545e622b045c/protocol/src/contracts/vesting/AlignerzVesting.sol#L613) and [bidding](https://github.com/dualguard/2025-11-alignerz-taridoku/blob/bca8c6f4be40b274a1c40cd2016a545e622b045c/protocol/src/contracts/vesting/AlignerzVesting.sol#L885) conditions. 
So, for every NFT that represents a TVS, `vesting.allocationOf(nftId).token` is set to the project token (the TVS token). To deploy the dividend distributor, this same token is passed as `_token` into the constructor. Hence `address(vesting.allocationOf(nftId).token) == address(token);`

The issue then lies in the [`A26ZDividendDistributor.getUnclaimedAmounts`](https://github.com/dualguard/2025-11-alignerz-taridoku/blob/bca8c6f4be40b274a1c40cd2016a545e622b045c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161). This line: 
```solidity
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
```
says that 
```init
If the NFT’s `allocation.token` matches the distributor’s configured token, return 0 and do not count any unclaimed amount for this NFT
```
Given the wiring above, that is backwards. TVS positions this distributor should pay dividends for are exactly those where
`allocation.token == token` but those are the ones being excluded. The condition is currently: 
```solidity
if (isThisExactlyTheTokenWeCareAbout(nftId)) {
    // ignore it
    return 0;
}
```
instead of 
```solidity
if (allocation.token != token) {
    // ignore foreign / irrelevant NFTs
    return 0;
}
```


### Internal Pre-conditions

1. Token check in `A26ZDividendDistributor.getUnclaimedAmounts` is flipped.

### External Pre-conditions

Nil

### Attack Path

1. Owner configures the dividend distributor
2. Owner tries to initialize dividends with `setUpTheDividends()` which calls `_setAmounts()` then `_setDividends()`
3. `_setAmounts()` computes totals where theyve funded the distributor with stable coin so `stablecoinAmountToDistribute > 0`
4. `getTotalUnclaimedAmounts()` iterates all NFTs
5. getUnclaimedAmounts() has the flipped token check
6. Result:
- `getUnclaimedAmounts(nftId)` returns 0 for every NFT
- `unclaimedAmountsIn[nftId]` remains 0
- `getTotalUnclaimedAmounts()` finally returns 0
7. `_setAmounts()` finishes with a zero denominator
8. `_setDividends()` now divides by zero leading to revert 

### Impact

High impact:
It bricks dividend distribution mechanism, returning 0 for every correct NFT. 
`getUnclaimedAmounts` returns 0 for every correct NFT (matching token) -> totalUnclaimedAmounts becomes 0 and finally `_setDividends does ... / totalUnclaimedAmounts` -> division by zero -> hard revert.

High likelihood as this is guaranteed in a normal configuration

### PoC

_No response_

### Mitigation

_No response_
  