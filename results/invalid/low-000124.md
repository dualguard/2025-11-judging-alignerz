# [000124] Burned NFTs orphan vesting allocations and exclude holders from future dividends
  
  ### Summary

AlignerzNFT allows privileged minters to burn NFTs without coordinating with AlignerzVesting’s vesting state or A26ZDividendDistributor’s dividend logic. When a TVS NFT is burned outside the vesting contract’s own flows, its associated vesting allocation remains in storage but becomes unreachable because all vesting operations require a valid NFT owner. Dividend calculations also skip such NFTs entirely. The impact is high, as a minter can permanently deny holders access to both vesting claims and future dividends for those TVSs, but the likelihood is low as this role is trusted leading to an overall Low severity

### Root Cause

.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. The system is deployed with at least two minters: the vesting proxy and the deployer EOA, as shown in deployment scripts where the deployer is made a minter and ownership is transferred.
2. A user holds a TVS NFT `nftId` with an active allocation in `AlignerzVesting` (`allocationOf[nftId]` set, and project-specific allocation non-empty).
3. The deployer EOA (or any other minter) calls:
   ```solidity
   nft.burn(nftId);
   ```
   This succeeds, as it only checks `onlyMinter` and `_exists(nftId)`, and does not interact with `AlignerzVesting`.
4. From this point:
   - `nft.extOwnerOf(nftId)` will revert (or fail), so:
     - `claimTokens(projectId, nftId)` reverts at `extOwnerOf(nftId)`.
     - `mergeTVS()` / `splitTVS()` involving `nftId` also revert on `extOwnerOf(nftId)`.
   - In `A26ZDividendDistributor`, `safeOwnerOf(nftId)` returns `(address(0), false)`, so `getTotalUnclaimedAmounts()` and `_setDividends()` skip `nftId` entirely when allocating dividends.
5. The user cannot claim remaining vested tokens or receive dividends for that TVS. The allocation data remains in storage but is effectively orphaned

### Impact

Low severity

### PoC

_No response_

### Mitigation

Consider ensuring that only the vesting contract can burn TVS NFTs and that burns are always coordinated with vesting and dividend state:
  