# [000454] claimDividends' vesting period may differ from its NFT's
  
  ### Summary

In the distributor contract, users earn dividends based on the NFT they won. It is based both on the amount and on the vesting period. While the amount is correctly calculated, the vesting period is different from the one users bid on. They will earn their dividends based on the vesting period set in the contract (should be 1 month), even if they won their bid with a vesting period higher than that.

### Root Cause

`A26ZDividendDistributor` calculates dividends vesting period based on the vesting contract, but distributes dividends based on its own variable.

### Impact

Users who set a higher vesting period when betting will win bids but earn their dividends faster

### Mitigation

When claiming dividends, use the vesting period of the nft instead of the vesting period in the `A26ZDividendDistributor` contract.
  