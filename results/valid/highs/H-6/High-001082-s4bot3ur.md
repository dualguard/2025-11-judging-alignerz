# [001082] Dividend Distributor Becomes Permanently Bricked Due to OOG When Iterating All NFTs
  
  ### Summary

The unbounded iteration in `A26ZDividendDistributor::getTotalUnclaimedAmounts()` will cause a permanent Out Of Gas (OOG) failure, preventing dividend amounts from ever being set.
This will cause all dividend distributions to fail for every project, as _setAmounts() depends on this function and will revert once NFT count grows large.

Because AlignerzNFT is global across all projects, NFT IDs increase forever. Eventually, `getTotalUnclaimedAmounts()` becomes non-executable due to OOG, so no project will be able to initialize dividends, causing total loss of functionality.

### Root Cause

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L129

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L216

The above loops are non-bounded loops over all minted NFTs.
Since NFT IDs increase globally across all projects, this loop grows indefinitely.
When len becomes large enough, the loop cannot fit into a single block’s gas limit and execution always reverts.

This permanently bricks `_setAmounts()` and prevents any future dividend setup.

### Internal Pre-conditions

1. The owner must transfer the dividend amount to the deployed `A26ZDividendDistributor` contract.
2. Owner must to call `A26ZDividendDistributor::setUpTheDividends` or `A26ZDividendDistributor::setAmounts` 

### External Pre-conditions

1. `AlignerzVesting` must be deployed, and at least one project must be created.
2. Users must submit bids for that project, the project owner must finalize all bids, and many users(more than 200) must claim their NFTs.
3. An attacker (or any user) must split their TVS into multiple NFTs, increasing the global NFT count or or alternatively, the NFT count grows naturally as more projects are launched
4. The project owner must announce dividends and deploy an instance of `A26ZDividendDistributor` with valid parameters for that specific project.

### Attack Path

1. Users participate in **Project A** by placing bids, and NFTs are minted as expected.
2. The Vesting Owner finalizes the bids, and users begin claiming their NFT-based TVS allocations.
3. An attacker (or any user) repeatedly splits their TVS into many TVSs, drastically increasing the global NFT count — or alternatively, the NFT count grows naturally as more projects are launched.
4. The Project A Owner initiates a dividend distribution and deploys `A26ZDividendDistributor,` transfers the dividend tokens, and calls `setUpTheDividends()`.      
    - During this call, the contract executes getTotalUnclaimedAmounts(), which loops over all minted NFT IDs.     
    - As the NFT count becomes large, the loop exceeds the gas limit, causing the transaction to revert permanently.

### Impact

Dividend distribution cannot be set up because the contract always reverts when iterating through all NFT's (if too many NFTs are minted). This makes dividend distribution impossible, and all the tokens sent to the distributor get stuck until the owner manually withdraws them. The team must redeploy a fixed distributor contract for dividends to work.

### PoC

_No response_

### Mitigation

One possible mitigation is to track each project’s `unclaimedAmount` directly in the Vesting contract.
Whenever a user wins a bid and claims their NFT or when the project owner allocates TVS for KOLs, the vesting contract should increase the project’s `unclaimedAmount` by the corresponding token amount.
Similarly, whenever an NFT owner claims their vested project tokens, the vesting contract should decrease the `unclaimedAmount` globally and per user accordingly.

When dividends are announced and a user calls claim in the Dividend Distributor, the distributor should read from the Vesting contract:
- The **total unclaimed amount** for the entire project
- The **individual unclaimed amount** for each NFT
The dividend share can then be calculated as:

```solidity
(user_unclaimed_amount / total_unclaimed_amount) * totalDividendAmount
```

  