# [001019] DOS on claimTokens() due to requiring the claimable amount > 0 for all flow indexes
  
  ### Summary

Claiming tokens implementation requires that the amount to be claimed for all flows to be greater than 0. Otherwise, it reverts, blocking the TVS holder to claim tokens from all flows. 

### Root Cause

`claimTokens()` calls [getClaimableAmountAndSeconds()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L958) in a loop to calculate the claimable amount for all flows a TVS has. If, for any reason, the claimable amount for a flow is 0, the tx [reverts](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L994). 

```solidity
    function claimTokens(uint256 projectId, uint256 nftId) external {
//...
        uint256 nbOfFlows = allocation.vestingPeriods.length;

        uint256 flowsClaimed;
        for (uint256 i; i < nbOfFlows; i++) { // @audit loop all flows
            if (allocation.claimedFlows[i]) {
                flowsClaimed++;
                continue;
            }
@>            (uint256 claimableAmount, uint256 claimableSeconds) = getClaimableAmountAndSeconds(allocation, i); 

//...
}

    function getClaimableAmountAndSeconds(Allocation memory allocation, uint256 flowIndex) public view returns(uint256 claimableAmount, uint256 claimableSeconds) {
        uint256 secondsPassed;
        uint256 claimedSeconds = allocation.claimedSeconds[flowIndex];
        uint256 vestingPeriod = allocation.vestingPeriods[flowIndex];
        uint256 vestingStartTime = allocation.vestingStartTimes[flowIndex];
        uint256 amount = allocation.amounts[flowIndex];
        if (block.timestamp > vestingPeriod + vestingStartTime) {
            secondsPassed = vestingPeriod;
        } else {
            secondsPassed = block.timestamp - vestingStartTime;
        }


        claimableSeconds = secondsPassed - claimedSeconds;
        claimableAmount = (amount * claimableSeconds) / vestingPeriod;
@>        require(claimableAmount > 0, No_Claimable_Tokens()); // @audit if claimable amount for this `flowIndex` is 0, revert
        return (claimableAmount, claimableSeconds);
    }
```


 _Note: For context, a `flow` can be defined as a single vesting stream. Each vesting stream has a collection of information associated with it: vesting period, amount, vesting start time, etc._ 

Users can split and merge TVS as they desire (as long as the vesting token is the same). 

By spliting a TVS, the [allocation.amount](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1132) for the resulting TVSs is set according to split percentages:

```solidity
    function _computeSplitArrays(..., uint256 percentage, ...)
//... 
        alloc.token = allocation.token;
        for (uint256 j; j < nbOfFlows;) {
@>            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
            alloc.vestingPeriods[j] = baseVestings[j];
//...
```
Then, when  2 or more TVSs are merged, their 'flows' are [linked](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1040-L1044) under a single TVS.
```solidity
    function _merge(Allocation storage mergedTVS, uint256 projectId, uint256 nftId, IERC20 token) internal returns (uint256 feeAmount) {
//...
        uint256 nbOfFlowsTVSToMerge = TVSToMerge.amounts.length;
        for (uint256 j = 0; j < nbOfFlowsTVSToMerge; j++) {
            uint256 fee = calculateFeeAmount(mergeFeeRate, TVSToMerge.amounts[j]);
@>            mergedTVS.amounts.push(TVSToMerge.amounts[j] - fee);
@>            mergedTVS.vestingPeriods.push(TVSToMerge.vestingPeriods[j]);
@>            mergedTVS.vestingStartTimes.push(TVSToMerge.vestingStartTimes[j]);
@>            mergedTVS.claimedSeconds.push(TVSToMerge.claimedSeconds[j]);
@>            mergedTVS.claimedFlows.push(TVSToMerge.claimedFlows[j]);
            feeAmount += fee;
        }
```

The resulting TVSs can have different flows with different parameters (amounts, claimable seconds left and vesting period). 
This increases the changes that when tokens are claimed, the `claimableAmount` for a flow to round down to  `0`. 

In the worst case scenario, even after the tokens are fully vested, the `claimableAmount` will round down to `0`, blocking the TVS holder to claim the tokens from the other flows. 


### Internal Pre-conditions

- A TVS with 2 or more flows. 

### External Pre-conditions

None

### Attack Path

The vulnerable path is straight forward as explained in the Root cause. 
1. Alice holds a TVS with 2 or more flows
2. Alice calls `claimTokens()` to claim the tokens from the vested flows
3. `getClaimableAmountAndSeconds()` is called and `claimableAmount` for any flow rounds down to `0`. 
Due to the `require` check, the tx reverts:
```solidity
 function getClaimableAmountAndSeconds(Allocation memory allocation, uint256 flowIndex) public view returns(uint256 claimableAmount, uint256 claimableSeconds) {
//...

require(claimableAmount > 0, No_Claimable_Tokens())
```
4. Alice can't claim the tokens from the TVS. 


### Impact

Vested tokens can't be claimed. Tokens are 

### PoC

_No response_

### Mitigation

The simplest solution is to remove the `require` check from `getClaimableAmountAndSeconds()`. 
Keep in mind that, due to rounding down, the `claimedSeconds` may be increases by `claimableSeconds` but user to receive 0 amount. 
The rounding down happens with or without the `require` check. 
  