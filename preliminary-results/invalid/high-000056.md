# [000056] Time-Ratio Formula Causes Direct Loss of User Funds on Reward Top-Ups
  
  ## Summary

The `A26ZDividendDistributor` contract cannot properly handle multiple distribution periods as described in the whitepaper. The whitepaper promises quarterly dividend distributions with recalculation based on current unreleased tokens, but the contract uses a single `startTime` and `claimedSeconds` that cannot track multiple vesting periods. This causes users to either receive incorrect dividend amounts or be unable to claim when new distributions are added.

Affected Functions: [`_setDividends()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L214), [`claimDividends()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L187), [`setStartTime()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L98), [`setUpTheDividends()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L111)

Root Cause: Architectural flaw - single timeline design cannot support adding rewards mid-vesting. The formula `claimableAmount = totalAmount * claimableSeconds / vestingPeriod` assumes all rewards existed from start.

## Vulnerability Details

### Whitepaper Promise

From Section 5.2 - "Vesting Reward Weird Distribution":

> "5% of our profit will be redistributed directly to our TVS holders, proportionally to their locked tokens. Vesting rewards will be calculated on a quarterly basis but will be dripped to TVS holders by the second."

The whitepaper provides an example showing monthly recalculation:

| Month | TVS A Unreleased | User Reward |
| ----- | ---------------- | ----------- |
| 1     | 9,000 tokens     | $543,248    |
| 2     | 8,000 tokens     | $659,358    |
| 3     | 7,000 tokens     | $909,090    |

Key observation: The rewards change each month because they're recalculated based on current unreleased tokens. This is NOT a single distribution dripped over time - it's multiple separate distributions.

### Contract Design

The contract has a single vesting timeline:

```solidity
uint256 public startTime;
uint256 public vestingPeriod;

struct Dividend {
    uint256 amount;
    uint256 claimedSeconds;
}

mapping(address => Dividend) dividendsOf;
```

When dividends are set:

```solidity
function _setDividends() internal {
    uint256 len = nft.getTotalMinted();
    for (uint i; i < len;) {
        (address owner, bool isOwned) = safeOwnerOf(i);
        if (isOwned) {
            dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
            // ↑ Uses += to accumulate
        }
        unchecked {
            ++i;
        }
    }
}
```

### The Problem: Cannot Handle Multiple Periods

The contract's design fundamentally breaks when the admin performs a "Top-Up" - adding new quarterly profits to the reward pool after vesting has started. This is a core feature promised in the whitepaper ("calculated on a quarterly basis"), but the time-ratio formula causes early claimers to lose funds.

#### The Top-Up Scenario

What is a Top-Up?

A "Top-Up" is when the admin adds new funds (quarterly profits) into the reward pool after vesting has already started. The whitepaper promises this will happen every quarter.

The Math Breakdown:

```solidity
// Current claiming formula
claimableAmount = dividendsOf[user].amount * claimableSeconds / vestingPeriod
```

This formula assumes `dividendsOf[user].amount` existed from the beginning. When new funds are added mid-vesting, early claimers are penalized.

Concrete Example:

```
Month 0 (Start):
- Admin allocates $1,000 to Alice
- dividendsOf[Alice].amount = $1,000
- vestingPeriod = 10 months
- claimedSeconds = 0

Month 5 (Halfway - Alice Claims):
- Time passed: 50%
- Claimable: $1,000 * 50% = $500
- Alice receives: $500 ✓
- dividendsOf[Alice].claimedSeconds = 5 months
- dividendsOf[Alice].amount = $1,000 (unchanged)

Month 6 (Top-Up - Admin Adds Quarterly Profits):
- Admin calls setUpTheDividends()
- New allocation: $1,000
- dividendsOf[Alice].amount += $1,000
- dividendsOf[Alice].amount = $2,000 (total)
- claimedSeconds = 5 months (unchanged)

Month 10 (End - Alice Claims Remaining):
- Time passed: 10 months
- Time remaining: 10 - 5 = 5 months (50%)
- Claimable: $2,000 * 50% = $1,000
- Alice receives: $1,000

Total Alice Received: $500 + $1,000 = $1,500
Total Alice Was Allocated: $1,000 + $1,000 = $2,000
LOSS: $500 (25% of her rewards!)
```

Why This Happens:

The formula applies the new $1,000 only to the remaining 5 months (Months 6-10). Alice already "used up" the first 5 months of time, so she gets ZERO of the new $1,000 for that period. The contract effectively steals 50% of her second allocation.

## Impact

The whitepaper explicitly promises quarterly distributions:

- "Vesting rewards will be calculated on a quarterly basis"
- Example shows monthly recalculation with changing amounts

The contract cannot deliver this functionality.

### Direct Loss of User Funds

```solidity
// This formula assumes 'amount' existed from startTime
claimableAmount = amount * claimableSeconds / vestingPeriod
```

When new funds are added via Top-Up:

- Early claimers lose a portion of new rewards proportional to time already claimed
- A user who claimed 50% before a Top-Up loses 50% of the new allocation
- The "lost" funds remain stuck in the contract or get redistributed incorrectly

  