# [000815] A26ZDividendDistributor re-distributes the unclaimed amounts from other users, causing incorrect accounting and insolvency
  
  ### Summary


The `_setAmounts()` function in `A26ZDividendDistributor` incorrectly uses the entire stablecoin balance of the contract (`balanceOf(address(this))`) to calculate dividend distribution. This includes unclaimed dividends from previous rounds, causing the system to redistribute old unclaimed funds along with new deposits. 

As a result, users receive significantly more dividends than the admin deposited, leading to contract insolvency where later claimers cannot withdraw their funds.


### Root Cause


The contract `A26ZDividendDistributor` is intended to distribute protocol profits to A26Z TVS holders: Every quarters (or fixed schedule), the admin will distribute the profits through this contract. The admin will do that by directly transferring stablecoins into the distributor contract, and then call `setUpTheDividends()` to distribute the funds.

The flaw is that there may be unclaimed stablecoins from the previous distributions, but `_setAmounts()` fail to distinguish that. As a result, the unclaimed portion will be double-distributed, causing users to receive more than they should, and causing protocol insolvency.

During `_setAmounts()`, the amount to distribute is recorded:

```solidity
function _setAmounts() internal {
    stablecoinAmountToDistribute = stablecoin.balanceOf(address(this));  // @audit this includes unclaimed balance from previous distributions
    totalUnclaimedAmounts = getTotalUnclaimedAmounts();
    emit amountsSet(stablecoinAmountToDistribute, totalUnclaimedAmounts);
}
```

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L207

And then in `_setDividends()`, these amounts are distributed to users, pro-rata of their unclaimed amounts in the TVS:

```solidity
function _setDividends() internal {
    uint256 len = nft.getTotalMinted();
    for (uint i; i < len;) {
        // ...
        if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
        // ...
    }
}
```

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L218

Because `stablecoinAmountToDistribute` mistakenly includes the already distributed amount, those amount is effectively distributed twice. This causes everyone to be entitled to more than what they should, and also causing insolvency to the distributor.

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

1. In the first quarter, the admin deposits 100,000 USDC and distributes it equally among two users (Alice and Bob), each receiving 50,000 USDC in dividends. 

2. Only Alice claims the 50,000 USDC. Bob has 50,000 USDC unclaimed, which remains in the contract.

3. In the second quarter, the admin deposits another 100,000 USDC and distributes it equally among two users (Alice and Bob), each **should** be receiving 50,000 USDC in dividends. 

4. However, the contract now holds 150,000 USDC (the new 100,000 USDC and Bob's unclaimed 50,000 USDC). But the contract thinks it should distribute its whole balance of 150,000 USDC, and gives Alice and Bob 75,000 USDC each in dividends instead.

Alice is now entitled to claiming 75,000 USDC, and for Bob, 125,000 USDC, even though the contract only holds 150,000 USDC. Both users have their rewards inflated, and whoever claims first can claim the inflated amount and cause loss for the other user.

Note that this creates an incentive for Bob to not claim, because Bob can weaponize the bug and claim 125,000 rather than 100,000. This makes it an exploitable bug.


### Impact


- Funds are wrongfully distributed, users will be able to claim more than what they should be able to
- Insolvency when the contract run out of funds to distribute.

### PoC

_No response_

### Mitigation


`_setAmounts()` and `_setDividends()` must separately track the unclaimed stablecoin amounts from the newly arrived amounts.

```solidity
// Add state variable
uint256 public totalUnclaimedDividends;

function _setAmounts() internal {
    uint256 currentBalance = stablecoin.balanceOf(address(this));

    // NEW: Only distribute NEW funds (exclude unclaimed from previous rounds)
    stablecoinAmountToDistribute = currentBalance - totalUnclaimedDividends;

    totalUnclaimedAmounts = getTotalUnclaimedAmounts();
    emit amountsSet(stablecoinAmountToDistribute, totalUnclaimedAmounts);
}

function _setDividends() internal {
    uint256 len = nft.getTotalMinted();
    uint256 allocatedThisRound = 0;

    for (uint i; i < len;) {
        (address owner, bool isOwned) = safeOwnerOf(i);
        if (isOwned) {
            uint256 allocation = unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts;
            dividendsOf[owner].amount += allocation;
            allocatedThisRound += allocation;
        }
        unchecked {
            ++i;
        }
    }

    // Track total unclaimed for next round
    totalUnclaimedDividends += allocatedThisRound;

    emit dividendsSet();
}

function claimDividends() external {
    address user = msg.sender;
    uint256 totalAmount = dividendsOf[user].amount;
    uint256 claimedSeconds = dividendsOf[user].claimedSeconds;
    uint256 secondsPassed;

    if (block.timestamp >= vestingPeriod + startTime) {
        secondsPassed = vestingPeriod;
        dividendsOf[user].amount = 0;
        dividendsOf[user].claimedSeconds = 0;
    } else {
        secondsPassed = block.timestamp - startTime;
        dividendsOf[user].claimedSeconds += (secondsPassed - claimedSeconds);
    }

    uint256 claimableSeconds = secondsPassed - claimedSeconds;
    uint256 claimableAmount = totalAmount * claimableSeconds / vestingPeriod;

    // Decrease unclaimed tracker when user claims
    totalUnclaimedDividends -= claimableAmount;

    stablecoin.safeTransfer(user, claimableAmount);
    emit dividendsClaimed(user, claimableAmount);
}
```

  