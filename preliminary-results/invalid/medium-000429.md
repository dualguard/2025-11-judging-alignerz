# [000429] The use of ``endTimeHash`` is useless.
  
  ### Summary

The use of ``endTimeHash`` is useless.

### Root Cause

As per the ``whitepaper``:

> To prevent last-minute bid manipulation, bidding window will be hashed.
Ps: bidding window is a period of time that starts as soon as the pools are filled.
After the bidding ends, we will reveal the exact input used to generate the hash, ensuring
fairness.
The input format will be intentionally unconventional to prevent prediction. Ex: "OnE
hOUR :+ tHirtY minutEs +-@ 20 fIve sECoNDS = 1h:30m:25s" which would appear
publicly on the bidding page as a unique string.
Ex:"7d1ae2ed5171c06d670f4aa30b595712bc81ba0e22a8811fa9eedc5bb3a236fd".

> The bidding time will vary between 0h:00m:00s and 5h:55m:55s, randomly drawn in a lottery-
like system. Bidding results are dynamic and visible in real-time.

> This ensures no insiders, or last-minute bidders can game the system.

https://drive.google.com/file/d/1xN5bYUPd_BkBMtoRruHEO1CBUx0vBiit/edit?pli=1

The ``endTimeHash`` is a encrypted ``hash`` to prevent ``insiders or last minute bidders`` by ``encrypting`` the ``endTime``. But if we look at ``placeBid()`` function:
```solidity
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
....
        biddingProject.bids[msg.sender] =
            Bid({amount: amount, vestingPeriod: vestingPeriod});
        biddingProject.totalStablecoinBalance += amount;

        emit BidPlaced(projectId, msg.sender, amount, vestingPeriod);
    }
```
``biddingProject.endTime`` is also used.

In the ``launchBiddingProject()``, ``endTime`` is set with ``endTimeHash``.
```solidity
    function launchBiddingProject(
        address tokenAddress,
        address stablecoinAddress,
        uint256 startTime,
        uint256 endTime,
        bytes32 endTimeHash,
        bool whitelistStatus
    ) external onlyOwner {
        require(tokenAddress != address(0), Zero_Address());
        require(stablecoinAddress != address(0), Zero_Address());
        require(startTime < endTime, Starttime_Must_Be_Smaller_Than_Endtime());

        BiddingProject storage biddingProject = biddingProjects[biddingProjectCount];
        biddingProject.token = IERC20(tokenAddress);
        biddingProject.stablecoin = IERC20(stablecoinAddress);
        biddingProject.startTime = startTime;
        biddingProject.endTime = endTime; // here
        biddingProject.poolCount = 0;
        biddingProject.endTimeHash = endTimeHash; // here
        isWhitelistEnabled[biddingProjectCount] = whitelistStatus;
        emit BiddingProjectLaunched(biddingProjectCount, tokenAddress, stablecoinAddress, startTime, endTimeHash);
        biddingProjectCount++;
    }
```
This implementation makes the entire ``hash for fairness`` mechanism completely pointless. Bidders can see the exact end time from the moment the auction starts, which is exactly what the original system was designed to prevent.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Anyone can decode the ``launchBiddingProject()`` transcation and read the ``endTime``.

### Impact

1. ``endTimeHash`` is not implemented properly. 
2. There are other incentives in a ``biddingProject`` like:
> We also decided to add additional incentives such as first pool & last pool winners will keep their
allocation & get refunded.

Anyone knowing the ``endTime`` can place ``last second bid`` to be at the top of the first pool by setting ``vestingPeriod`` the ``highest``. As, bidders are ordered by ``vestingPeriod``. This is unfair for honest ``bidders``  and breaks a core functionality.

### PoC

_No response_

### Mitigation

Implement ``endTimeHash`` and ``endTime`` differently if you want to keep ``bidding window`` anonymous.

Example Implementation:
```solidity
// During bidding - only check start time
require(block.timestamp >= biddingProject.startTime, "Bidding not started");
require(!biddingProject.endTimeRevealed, "Bidding ended");

// After bidding ends - owner reveals the actual end time
function revealEndTime(uint256 projectId, uint256 actualEndTime, string memory unconventionalString) external onlyOwner {
    bytes32 calculatedHash = keccak256(abi.encodePacked(unconventionalString));
    require(calculatedHash == biddingProjects[projectId].endTimeHash, "Hash mismatch");
    require(actualEndTime >= biddingProjects[projectId].startTime, "Invalid end time");
    
    biddingProjects[projectId].actualEndTime = actualEndTime;
    biddingProjects[projectId].endTimeRevealed = true;
}
```
  