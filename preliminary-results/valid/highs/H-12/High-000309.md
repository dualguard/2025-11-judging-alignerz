# [000309] Rewards are allocated to different TVS token holders instead of to TVS token holders which is the same as the token on the dividend contract
  
  ### Summary

`A26ZDividendDistributor.sol` has the main function to distribute rewards to TVS holders, more precisely to TVS token holders that are the same as the token set in the constructor.

```solidity
constructor(address _vesting, address _nft, address _stablecoin, uint256 _startTime, uint256 _vestingPeriod, address _token) Ownable(msg.sender) {
        --- rest of code ---
        token = IERC20(_token);
    }
```

But in `getUnclaimedAmounts()` , there is wrong check. Instead check for TVS holder that have same token on `A26ZDividendDistributor.sol` and calculating the portion of rewards, the check returns 0 directly if the tokens are the same.

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
        --- rest of code ---
   }
```

This has an impact on TVS holders who have tokens that are the same as the dividend contract, not receiving rewards, but instead the opposite, TVS holders who have tokens that are not the same as the dividend contract, receive the  rewards.

### Root Cause

In [A26ZDividendDistributor.sol:141](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141) there is wrong check for token inside the TVS 

### Internal Pre-conditions

-

### External Pre-conditions

-

### Attack Path

1. Owner wants to distribute rewards through `A26ZDividendDistributor` contract for `Alignerz256` token holder on their TVS
2. Owner sent the reward to `A26ZDividendDistributor` contract
3. Owner call `setUpTheDividends()`
4. As the end, who receive the reward are non `Alignerz256` token holder on their TVS because of the check explained above

### Impact

This has an impact on TVS holders who have tokens that are the same as the dividend contract, not receiving rewards, but instead the opposite, TVS holders who have tokens that are not the same as the dividend contract, receive the  rewards.

### PoC

_No response_

### Mitigation

Consider change check to :

```diff
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
--      if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
++      if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;
        --- rest of code ---
   }
```
  