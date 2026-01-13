# [001018] TVSs may be split right before the TVS is sold. Buyer gets less value than expected
  
  ### Summary

A `splitTVS()` tx may be executed right before the TVS is sold on a NFT marketplace. 
Since the original TVS id is not burned in a split, the buyer gets less value than expected. 

### Root Cause

`splitTVS()` allows a TVS holder to divide the future token vesting into multiple TVSs according to desired `percentages`. 
When a holder divides his TVS into multiple ones, the [original TVS ID is recycled](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1086) (not burned). The [original TVS amount](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1132) is set according to the first percentage value from `percentages[]` argument array.

```solidity
    function splitTVS(
        uint256 projectId,
        uint256[] calldata percentages,
        uint256 splitNftId
    ) external returns (uint256, uint256[] memory) {
//...
        uint256 nbOfTVS = percentages.length;

        // new NFT IDs except the original one
        uint256[] memory newNftIds = new uint256[](nbOfTVS - 1);

        // Allocate outer arrays for the event
        Allocation[] memory allAlloc = new Allocation[](nbOfTVS);

        uint256 sumOfPercentages;
        for (uint256 i; i < nbOfTVS;) {
            uint256 percentage = percentages[i];
            sumOfPercentages += percentage;

@>            uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender); // @audit the original nftID is not burned
            if (i != 0) newNftIds[i - 1] = nftId;
            Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
//...
```

The problem is that a `splitTVS()` tx may be executed right before the TVS is sold, causing the buyer to receive  only a fraction of the value paid.  





### Internal Pre-conditions

- A TVS to be listed on a marketplace (or an exchange between 2 parties that involves a TVS and any other asset). 

### External Pre-conditions

None

### Attack Path

1. Alice hold TVS1 with  pending vesting tokens worth 1000 USD
2. Alice list her TVS1 an an marketplace for 900 USD
3. Bob buys the TVS1
4. Right before Bob's tx is received and proceessed, the L2 Sequencer receive Alice's `splitTVS()` tx. 
Since the original NFT ID is not burned by the `splitTVS()` operation, Bob receive just a percentage of the value of the original TVS (before split). 

In a more complex scenario, a malicious user can create malicious wallet that holds TVS and convince users to buy the TVS. When buyer executes the buy tx, the TVS is first split and then is sent to buyer address. Buyer receive a worthless TVS. 

### Impact

TVS buyers risk obtaining significantly less value than the amount they paid.

### PoC

_No response_

### Mitigation

Consider burning the original TVS ID when calling `splitTVS()` and mint new IDs, one for each value from the `percentages[]` argument array. 
  