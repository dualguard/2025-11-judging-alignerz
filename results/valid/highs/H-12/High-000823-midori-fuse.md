# [000823] Dividends wrongly excludes the intended token and will always pay the wrong NFTs
  
  ### Summary

Per the A26Z tokenomics, holders of the A26Z TVS will be eligible for profit distribution in USDC. The contract `A26ZDividendDistributor` is intended to serve that purpose: Every quarters (or fixed schedule), the admin will distribute the profits through this contract.

However, `getUnclaimedAmounts()` currently has a wrong check that excludes the intended NFTs and includes the unintended NFTs. This will cause wrong NFTs to be rewarded and correct NFTs to be not rewarded.

### Root Cause


In `getUnclaimedAmounts()`, there is a check as follow:

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
	if (address(token) == address(vesting.allocationOf(nftId).token)) return 0; // @audit returns 0 if the token *is* A26Z
```

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141

This check returns early if the TVS **is** for the A26Z vesting, rather than is not.

The flow for admin to distribute profits to TVS holders is as follow:
- Transfer the profit stablecoins directly to the Diviend Distributor contract
- Call `setUpTheDividends()`, which will set up the distribution

`getUnclaimedAmounts()` is used to calculate the unclaimed amounts of the i-th NFT. However, the function returns 0 **if** the NFT is an A26Z TVS, while it is actually these exact NFTs that should be entitled to the profits.

This wrongly causes `unclaimedAmountsIn` for all the non-A26Z NFTs, and those only, to be non-zero. All the incorrect NFTs will be rewarded, and all the correct NFTs will not be rewarded.

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

- Admin calls `setUpTheDividends()` to set up profit distribution
- The contract wrongly rewards all the NFTs that are **not** of A26Z TVSes, rather than those of A26Z TVSes


### Impact

All of the correct NFTs will be excluded from reward distrbution. All of the wrong NFTS will be included in reward distribution

### PoC

_No response_

### Mitigation


The check should be

```solidity
if (address(token) != address(vesting.allocationOf(nftId).token)) return 0; // @audit NOT ==
```

  