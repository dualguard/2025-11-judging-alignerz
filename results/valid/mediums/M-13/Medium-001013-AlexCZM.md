# [001013] Dividend distribution implementation doesn't match the whitepaper details
  
  ### Summary

The dividend distribution implementation doesn't match the formula used in [whitepaper](https://drive.google.com/file/d/1xN5bYUPd_BkBMtoRruHEO1CBUx0vBiit/edit) chapter `5.2. Vesting Reward Weird Distribution`. 
In whitepaper the dividends are distributed based on locked, unreleased tokens; in the implementation the vested but not claimed tokens are considered for dividend distribution.

### Root Cause


According to whitepaper's `5.2 Vesting Reward Weird Distribution`: 

```
That's why 5% of our profit will be redistributed directly to our TVS holders, proportionally to 
their **locked** tokens.
[...]

Example:  
TVS A hold 10 000 $A26Z vested for 10 months. 
TVS B hold 1 000 $A26Z vested for 10 months. 
TVS C hold 10 000 $A26Z vested for 3 months. 
 
Let’s consider the first wave of distribution is worth 1 000 000 $S/month. For the sake of the 
example, let’s consider only 3 users secured a $A26Z TVS. 
```
| Month | TVS | Released Tokens $A26Z | Unreleased Tokens $A26Z | Total Vesting Rewards in $ | User Reward in $ |
|-------|-----|-------------------------|---------------------------|------------------------------|--------------------|
| 1     | A   | 1000                    | 9000                      | 1 000 000                    | 543 248.6268 $     |
| 1     | B   | 100                     | 900                       | 1 000 000                    | 54 324.86268 $     |
| 1     | C   | 3333                    | 6667                      | 1 000 000                    | 402 426.5105 $     |

#### 1. Whitepaper Spec 
- talks about `locked tokens`
- the formula used to reward the users consider the `Unreleased Tokens`; unreleased means tokens that are not vested yet. 

By example  a user's reward is calculated using the following fromula: 
`user's unreleased tokens` / `total unreleased tokens` * `total vesting rewards`. 
For user A: 
- `rewards = 9_000 / (9_000 + 900 + 6_667) * 1M = 9_000 / 16_567 * 1M = 543_248 USD`
UserA calculated rewards (543_248) matches the value from the table. 

#### 2. Code implementation
- calculates and saves the [unclaimedAmountsIn[]](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L153-L160) for each nftID
- saves [totalUnclaimedAmounts](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L209) 
- calculates the [user dividends](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L218) based on `userUnclaimed * totalRewards / totalUnclaimed` formula:
```solidity
dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts)
```

### Internal Pre-conditions

None

### External Pre-conditions

None 

### Attack Path

The vulnerable path is as explained in Root Cause. The code implementation dividend distribution follows a different formula than the one form the whitepaper. 

### Impact

Users suffer from inaccurate reward allocation, as the protocol distributes different rewards than those communicated beforehand

### PoC

_No response_

### Mitigation

Consider modifying the implementation to align with the whitepaper, or updating the whitepaper to accurately reflect the dividend distribution details.
  