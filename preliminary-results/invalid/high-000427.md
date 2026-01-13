# [000427] ``A whale`` can ``buy`` the whole pool allocation and get refunded due to the refund mechanism and profit from ``dividends``.
  
  ### Summary

``A whale`` can ``buy`` the whole pool allocation and get refunded due to the refund mechanism and profit from ``dividends``.

### Root Cause

As per whitepaper, 
> We also decided to add additional incentives such as first pool & last pool winners will keep their
allocation & get refunded.

<img width="647" height="247" alt="Image" src="https://github.com/user-attachments/assets/10c2a969-20ad-442d-953b-969a16dde5cd" />

https://drive.google.com/file/d/1xN5bYUPd_BkBMtoRruHEO1CBUx0vBiit/edit?pli=1

1. Let's say a whale ``Alice`` buys the whole ``pool1`` for ``220,000$`` with a very high vesting period(so that Alice can be at the top of the leaderboard. 
2. The vesting period can be set to highest possible number i.e ``(uint256).max / 2_592_000 * 2_592_000)`` ensure ``Alice`` gets to the top of the ``leaderboard`` as the ``leaderboard`` is based on ``vestingPeriods``.
3. ``Alice`` gets ``refunded`` and ``griefs`` the protocol and further ``griefs`` them by claiming ``quarterly`` dividends from the ``unclaimed`` amount for that large ``vesting`` period.

```solidity
./A26ZDividendDistributor.sol

    function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len;) {
            (address owner, bool isOwned) = safeOwnerOf(i);
            if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
            unchecked {
                ++i;
            }
        }
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
        stablecoin.safeTransfer(user, claimableAmount);
        emit dividendsClaimed(user, claimableAmount);
    }
```
Dividends are based on ``unclaimed amount`` and amount becomes ``claimable by second`` but due to larger vesting period, It will take large amount of time for the amount to be fully claimable. What if ``vesting time`` is set to ``(uint256).max / 2_592_000 * 2_592_000``.

 And yes, ``placeBid()`` has no max vesting period amout limit set.

```solidity
./AlignerzVesting.sol

    function placeBid(uint256 projectId, uint256 amount, uint256 vestingPeriod) external {
        if (isWhitelistEnabled[projectId]) {
            require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted());
        }
        require(projectId < biddingProjectCount, Invalid_Project_Id());
        require(amount > 0, Zero_Value());

        BiddingProject storage biddingProject = biddingProjects[projectId];
        require(
            block.timestamp >= biddingProject.startTime && block.timestamp <= biddingProject.endTime && !biddingProject.closed,
            Bidding_Period_Is_Not_Active()
        );
        require(biddingProject.bids[msg.sender].amount == 0, Bid_Already_Exists());

        require(vestingPeriod > 0, Zero_Value());

        require (vestingPeriod < 2 || vestingPeriod % vestingPeriodDivisor == 0, Vesting_Period_Is_Not_Multiple_Of_The_Base_Value());

        biddingProject.stablecoin.safeTransferFrom(msg.sender, address(this), amount);
        if (bidFee > 0) {
            biddingProject.stablecoin.safeTransferFrom(msg.sender, treasury, bidFee);
        }
        biddingProject.bids[msg.sender] =
            Bid({amount: amount, vestingPeriod: vestingPeriod});
        biddingProject.totalStablecoinBalance += amount;

        emit BidPlaced(projectId, msg.sender, amount, vestingPeriod);
    }
```

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

Mentioned in the Root Cause

### Impact

A ``whale`` can exploit the protocol functionality of the winner of first and last pool gets refunded in their favor and cause ``grief`` to the protocol and other ``bidders``.
Also, anyone can carry out this attack but a whale will cause the most damage to everyone.

### PoC

_No response_

### Mitigation

This stems from choosing winners based on vesting schedules, and incentive for being at top of the first and last pool. The incentive part needs to be removed.
  