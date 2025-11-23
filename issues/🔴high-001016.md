# [001016] TVS Holders dividends are calculated incorrectly due to wrong comparison operator
  
  ### Summary

[getUnclaimedAmounts()] uses wrong equality `==` operator instead of not equal `!=`. 
`totalUnclaimedAmounts` storage variable is set to wrong value and dividends for each holder are calculated incorrectly. 


### Root Cause

[getUnclaimedAmounts()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141) uses wrong equality `==` operator. 
It returns `0` if the 2 tokens matches and calculates and return the unclaimed amount if the tokens are different
```solidity
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;        @audi wrong comparison operator
//...
}
```


### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

Admin calls [setAmounts](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L117) -> [getTotalUnclaimedAmounts](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L209) -> [getUnclaimedAmounts](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L131C52-L131C71)
- `getUnclaimedAmounts` returns the wrong value and `totalUnclaimedAmounts` storage variable is updated to an incorect amount.
```solidity
    function _setAmounts() internal {
        stablecoinAmountToDistribute = stablecoin.balanceOf(address(this));
        totalUnclaimedAmounts = getTotalUnclaimedAmounts();                    // @audit it returns an incorrect value
        emit amountsSet(stablecoinAmountToDistribute, totalUnclaimedAmounts);
    }
```
- `totalUnclaimedAmounts` is used to calculate the amount to distribute to each holder
```solidity
    function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len;) {
            (address owner, bool isOwned) = safeOwnerOf(i);
@>            if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
//...
```

### Impact

Dividends for each holder are calculated incorrectly. 

### PoC

_No response_

### Mitigation

Update `getUnclaimedAmounts` :

```diff
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
-        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
+        if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;
//...
```
  