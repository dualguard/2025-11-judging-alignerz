# [001020] Splitting fee is calculated on the entire TVS allocation; the excess fee is transferred to treasury and is paid by  last users to claimTokens
  
  ### Summary


The splitting fee is calculated from the entire initial `amount` allocated to the TVS, rather than just the tokens still pending vesting. 
This results in an excessive fee being sent to the `treasury`, and the last users to claim may not receive their full tokens.


### Root Cause

Function `splitTVS()` calls [calculateFeeAndNewAmountForOneTVS()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1069) to calculate the `feeAmount` and the `newAmount` left after the fee is deducted. 
The problem is that the `calculateFeeAndNewAmountForOneTVS()` calculates the fee based on the full `amount` allocation and not on the the amount of tokens pending vesting. 

```solidity
// AlignerzVesting.sol
    function splitTVS(
        uint256 projectId,
        uint256[] calldata percentages,
        uint256 splitNftId
    ) external returns (uint256, uint256[] memory) {

        uint256[] memory amounts = allocation.amounts;
        uint256 nbOfFlows = allocation.amounts.length;
// @audit the full amount is passed to  calculate fee function
        (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);
        allocation.amounts = newAmounts;
        token.safeTransfer(treasury, feeAmount);
//...
}

// FeesManager.sol
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

The excess fee tokens are transferred to `treasury`; Last users to claim their vested tokens will not receive any tokens, paying all the excessive spliting Fee. 

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

Consider the following example, where the values were selected to make the flow easy to follow.
- Alice holds  `TVS1` with 1000 tokens committed (`allocation.amounts` = 1000)
- `vestingPeriods` = 1000 (seconds) and Alice claimed already 95% of it (corresponding to 950s out of 1000s). 
This means 50 tokens are left to be vested
- `splitFeeRate` = 1_000 (10%)
- Alice calls `splitTVS()` to split TVS1 in 2: `TVS2`  with 60% and `TVS3` with 40%. Even if first new TVS keeps the [original](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1086) TVS ID (nft id), let's name it `TVS2` to avoid confusion. 
1. the `feeAmount` = `10% * 1000  = 100` tokens. the new `allocation.amounts` is 900;  The `feeAmount = 100` is transferred to `treasury` 
2. 
- TVS2 amount, `allocation.amounts2` = `900 * 60% = 540` tokens
- TVS3 amount, `allocation.amounts3` = `900  * 40% = 360` tokens
 - the resto of `Allocation` data is identical to the `TVS1`
3. After another 50s both TVS2 and TVS3 amounts are fully vested; Alice calls `claimTokens()` for both TVSs. 
Claimable amount is calculated by [getClaimableAmountAndSeconds()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L958) function:
i. `claimableSeconds = secondsPassed - claimedSeconds;` => `claimableSeconds = 1000 - 950 = 50s`
ii. `claimableAmount = (amount * claimableSeconds) / vestingPeriod;` =>
- for TVS2: `claimableAmount = 540 * 50 / 1000 = 27`
- for TVS3: `claimableAmount = 360 * 50 / 1000 = 18`

Alice claimed from the splited TVS (TVS2 and 3): `27 + 18  = 45` tokens. It means that Alice paid for the split operation `50 - 45 = 5` tokens which represents 10% of the value of the TVS1 (the amount still pending vesting). 


In this example, `100` tokens are transferred to treasury instead of `5` tokens.
The vesting contract will be drained long before last user claim his last vested tokens. 

### Impact

An excessive  split fee is paid to treasury. The extra fee is paid by the last users to claim their last portion of  amounts vested. 

### PoC

_No response_

### Mitigation

Calculate the split fee from the amount of tokens pending vesting, not from the entire amount allocated to TVS. 
  