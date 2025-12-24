# [000455] Dividend distributor will distributes dividends for ALL projects
  
  ### Summary

As seen with the sponsor, the contract `A26ZDividendDistributor` is supposed to distribute a project's profits to the project's TVS holders. But the contract distributes dividends accross all NFT holders, who may hold TVS for other projects.

This will cause the distribution of dividends to go to all NFT holders instead of per-project NFT holders, rewarding users inequally.

### Root Cause

`A26ZDividendDistributor` lacks tracking NFT's project id for dividend distribution.

### Impact

The amount of dividends for project's NFT holders will be drastically reduced. Increased amount of undeserved rewards for other NFT holders.

### Mitigation

The functions to track the amounts to distribute should filter NFTs on the project id, as well as a variable to track the project id for each dividend distributor contract.
  