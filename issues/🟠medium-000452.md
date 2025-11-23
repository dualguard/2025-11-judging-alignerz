# [000452] When distributing dividends, getUnclaimedAmounts should calculate unclaimed available amounts
  
  ### Summary

`A26ZDividendDistributor::getUnclaimedAmounts` calculates the unclaimed amount of tokens of an NFT by substracting the total amount from the claimed amounts. This results in users being able to claim more dividends than what's really available to them.

### Root Cause

`getUnclaimedAmounts` should not calculate the amount of dividends are owed to a user. It should be the unclaimed **available** amounts.

### Attack Path

1. Alice mints an NFT. The NFT has `amount = 100_000_000` and a `vestingPeriod` of 1 month.
2. 2 weeks later, the project wants to distribute dividends to tokens holders. A new `A26ZDividendDistributor` contract is created.
3. The owner calls `setUpTheDividends`, which calculates and set the dividends for Alice to `amount - claimedAmount`, which is `100_000_000 - 0`.
4. Alice claims her whole package of dividends instead of having only the available amount

### Impact

Users claim more dividends than expected

### Mitigation

Instead of using `getUnclaimedAmounts`, use a function to get the unclaimed available amounts
  