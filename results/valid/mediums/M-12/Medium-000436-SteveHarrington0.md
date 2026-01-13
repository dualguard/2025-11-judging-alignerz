# [000436] claiming tokens will revert when allocation has multiple flows in some cases
  
  ### Summary


The `claimTokens` function enables users to claim the tokens corresponding to their allocations. Some allocations may consist of multiple flows due to the merging of multiple Token Vesting Schedules (TVSs). When a user calls `claimTokens` for a specific NFT ID, the function attempts to transfer the tokens associated with all flows to the user. However, because of a validation check in the `getClaimableAmountAndSeconds` function, the claim process will revert if any flow’s claimable amount is zero. This causes the entire claim transaction to fail, making it impossible for the user to claim the remaining tokens from other flows.



### Root Cause


The following snippet is part of the `claimTokens` function, which allows users to claim their eligible tokens. This function handles multiple vesting flows, as a Token Vesting Schedule (TVS) can represent several flows merged into one.

```solidity
for (uint256 i; i < nbOfFlows; i++) {
    if (allocation.claimedFlows[i]) {
        flowsClaimed++;
        continue;
    }
    (uint256 claimableAmount, uint256 claimableSeconds) = getClaimableAmountAndSeconds(allocation, i);
```

In this loop, the function iterates over each flow. If a flow has already been claimed, it increments the count and moves to the next flow. For unclaimed flows, it calls the `getClaimableAmountAndSeconds` function to determine how many tokens are currently claimable and the duration for which they can be claimed.

The `getClaimableAmountAndSeconds` function calculates these values as follows:

```solidity
claimableSeconds = secondsPassed - claimedSeconds;
claimableAmount = (amount * claimableSeconds) / vestingPeriod;
require(claimableAmount > 0, No_Claimable_Tokens());
```

Here, `claimableSeconds` is the difference between elapsed time and previously claimed seconds. The `claimableAmount` is prorated based on the vesting period. Importantly, the function reverts if the `claimableAmount` is zero.

This cause for reverting presents a limitation: if one flow's claimable amount is zero, the entire transaction reverts, preventing claims for other flows within the same transaction. This situation can occur when the `amount * claimableSeconds` is less than the `vestingPeriod`, which is a realistic scenario when the TVS is split into very small percentages or the user’s bid amount is small relative to a longer vesting period.



### Internal Pre-conditions



A user may bid for a very small amount combined with a longer vesting period, or alternatively, split a Token Vesting Schedule (TVS) into very tiny percentages, which can result in the condition where

$$
\text{amount} \times \text{claimableSeconds} < \text{vestingPeriod}
$$





### External Pre-conditions

None

### Attack Path



Consider the following scenario:

Bob owns a Token Vesting Schedule (TVS) with an ID of 10. This NFT ID 10 represents multiple vesting flows as follows:

```solidity
allocationOf[10] = {
    amounts: [1000 * 1e18, 6000, 5000 * 1e18],
    vestingPeriods: [180 days, 90 days, 365 days] 
    // in seconds -> [15,552,000, 7,776,000, 31,536,000]
};
```

Now, if the claimable amount for the second flow is calculated as:

```solidity
claimableAmount = (amount * claimableSeconds) / vestingPeriod
                = (6000 * x seconds) / 90 days = 0  // where x > 0, preferably 1 second
```

Since this calculation results in zero, the following check:

```solidity
require(claimableAmount > 0, No_Claimable_Tokens());
```

will cause the transaction to revert. As a result, users will be unable to claim tokens from the remaining two flows despite having claimable tokens there.



### Impact

Explained in Attack Path

### PoC

NA

### Mitigation

Remove the check `require(claimableAmount > 0, No_Claimable_Tokens());`.
  