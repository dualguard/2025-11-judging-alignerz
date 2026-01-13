# [000433] ``Partial Dividend claim`` will result in ``loss`` of dividend claim amount in the ``next period``.
  
  ### Summary

``Partial Dividend claim`` will result in ``loss`` of dividend claim amount in the ``next period`` as the ``claimDividends()`` can't handle partial claiming between multiple periods.

### Root Cause

In ``claimDividends()`` function:
```solidity
./A26ZDividendDistributor.sol

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
``Dividend`` is vested over a ``vesting period`` and claimable from ``startTime`` set in the contract.

> As per the protocol team, ``Dividend`` is 5% of our profit that will be redistributed directly to our TVS holders, every 3 months.

``Dividend`` is calculated and added by the ``_setDividends()`` function:
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
```

But in the ``claimDividends()``, ``dividendsOf[user].claimedSeconds`` is not ``reset`` when new ``dividend`` is distributed.

Let's say on first dividend distribution, 
``vestingPeriod = 100``
``dividendsOf[user].claimedSeconds = 90``
> User only claimed 90%

Now on second divided distribution, 
let's say ``vestingPeriod`` is again ``100``.
```solidity
        uint256 claimedSeconds = dividendsOf[user].claimedSeconds; // 90 persists
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
```
Now, even on full claim when:
```solidity
 if (block.timestamp >= vestingPeriod + startTime) {
            secondsPassed = vestingPeriod; // 100
            dividendsOf[user].amount = 0;
            dividendsOf[user].claimedSeconds = 0;
        } 
```
```solidity
        uint256 claimableSeconds = secondsPassed - claimedSeconds; // 100 - 90
        uint256 claimableAmount = totalAmount * claimableSeconds / vestingPeriod;
```
Only ``10%`` is claimable. And ``partial claim`` is not possible till secondsPassed = ``91``.
```solidity
   secondsPassed = block.timestamp - startTime;
            dividendsOf[user].claimedSeconds += (secondsPassed - claimedSeconds);
        }
        uint256 claimableSeconds = secondsPassed - claimedSeconds; // need 91 - 90 else revert
```

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. User need to partially claim dividend in one period.

### Impact

1. The user won't be able to claim ``dividends`` even if ``vesting time`` has fully passed.
2. ``Dividends`` won't be claimable by the second.
3. Temporary freezing of funds.

### PoC

``PoC`` where the user claims ``50%`` in one period and tries to claim ``100%`` in another period.
```solidity
    function test_pocPartialClaim() public {
        // First period Dividend Sent:
        uint256 dividendAmount = 100;
        uint256 claimedSeconds = 0; // persists for another period
        uint256 claimedSecondsTemp = claimedSeconds;
        uint256 vestingPeriod = 100;
        uint256 vestingStartTime = block.timestamp;
        uint256 secondsPassed;
        console.log(vestingStartTime);
        vm.warp(51);
        if (block.timestamp >= vestingPeriod + vestingStartTime) {
            secondsPassed = vestingPeriod;
            claimedSeconds = 0;
            dividendAmount = 0;
        } else {
            secondsPassed = block.timestamp - vestingStartTime;
            claimedSeconds += (secondsPassed - claimedSeconds);
        }
        uint256 claimableSeconds = secondsPassed - claimedSecondsTemp;
        uint256 claimableAmount = (dividendAmount * claimableSeconds) /
            vestingPeriod;
        console.log("First Partial Claimed Amount is", claimableAmount);

        // Second Period Dividend is added on top of first period
        vm.warp(50);
        dividendAmount += 100; // amount is added by 100, total 200 which much be claimed
        uint256 dividendAmountTemp = dividendAmount;
        vestingStartTime = block.timestamp;
        claimedSecondsTemp = claimedSeconds; // claimed seconds persists from first period
        vestingPeriod = 100;
        vm.warp(vestingPeriod + 1 + block.timestamp); // new period starts

        // full claim on the second one
        if (block.timestamp >= vestingPeriod + vestingStartTime) {
            secondsPassed = vestingPeriod;
            claimedSeconds = 0;
            dividendAmount = 0;
        }

        claimableSeconds = secondsPassed - claimedSecondsTemp;
        claimableAmount =
            (dividendAmountTemp * claimableSeconds) /
            vestingPeriod;
        console.log("Second Full Claimed Amount is: ", claimableAmount, "Instead of: ", dividendAmountTemp);
    }
```
Put the above ``PoC`` in ``AlignerzVestingProtocolTest.t.sol`` and run it using:
```solidity
forge test --match-test test_pocPartialClaim -vvv
```

### Mitigation

Reset ``dividendsOf[user].claimedSeconds`` after setting ``dividends`` for new ``period``.
  