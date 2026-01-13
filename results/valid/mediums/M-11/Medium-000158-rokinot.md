# [000158] Insolvency in case of admin seeding more funds to the distributor with unclaimed dividends
  
  The root lies within the _setDividends function of the distributor contract. Specifically, the line dividendsOf[owner].amount += ... uses an addition assignment operator (+=) to allocate dividend shares. This operator adds the newly calculated dividend amount to any value that already exists in a user's dividend balance. The flaw is that the function does not reset or clear the previous dividend allocations before calculating and assigning a new round. This means that if the owner calls the setUpTheDividends function before all dividends were claimed, the same dividend amounts are calculated and then stacked on top of the existing allocations, artificially inflating what each user is entitled to claim.

This leads to insolvency because the sum of total amounts can grow to exceed the actual funds held by the contract. For instance, if the owner calls setUpTheDividends twice, every user's claimable dividend amount is doubled, while the contract's stablecoin balance remains the same.

For example:
- The owner deposits 1,200,000 USDT into the A26ZDividendDistributor contract.
- The owner calls setUpTheDividends(). The contract  allocates a 400,000 USDT share to each of the three users 
- Partial Claims Occur: After 30 days of a 90-day vesting period, Alice and Bob each claim their vested portion of 133,333 USDT. Their total promised share in the contract (dividendsOf[user].amount) is not reset and remains at 400,000 USDT. The contract balance decreases to 933,334 USDT.
- The owner adds another 1,500,000 USDT to the contract, bringing the total balance to 2,433,334 USDT.
- The owner calls setUpTheDividends() again. Due to the bug, the contract calculates new shares based on the current balance and adds them to the users' existing allocations. Each user's promised share is now inflated to approximately 1,211,111 USDT. The contract is now insolvent, promising over 3.6M USDT while holding only 2.4M USDT.
- Once the dividend period ends, the user who hasn't claimed yet, withdraws his full inflated share of 1,211,111 USDT.
- Another user claims next, withdrawing her remaining inflated balance of 1,077,778 USDT.
- The last user, attempts to claim his remaining share. His transaction reverts with an insufficient funds error because the contract has been almost entirely drained by the first two claimants, leaving him with a significant loss.
  