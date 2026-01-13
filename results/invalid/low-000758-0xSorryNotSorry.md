# [000758] `setStablecoinAllocation` assumes non–fee-on-transfer stablecoins, causing underfunding if any of the known FOT stables (USDT) start implementing fees.
  
  
### Summary

The stablecoin allocation logic assumes that `totalStablecoinAllocation` tokens sent by the owner will fully arrive in the vesting contract, so using a fee-on-transfer stablecoin will underfund the pool and can later cause failed or unfair distributions.

### Root Cause

In `AlignerzVesting.sol:setStablecoinAllocation` the contract enforces `totalStablecoinAllocation == sum(stablecoinAmounts)` and then pulls `totalStablecoinAllocation` from the owner, assuming the contract’s balance will match the recorded allocations, which is false for fee-on-transfer tokens.

```solidity
function setStablecoinAllocation(
    uint256 rewardProjectId,
    uint256 totalStablecoinAllocation,
    address[] calldata kolStablecoin,
    uint256[] calldata stablecoinAmounts
) external onlyOwner {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    uint256 length = kolStablecoin.length;
    require(length == stablecoinAmounts.length, Array_Lengths_Must_Match());
    uint256 totalAmount;
    for (uint256 i = 0; i < length; i++) {
        address kol = kolStablecoin[i];
        rewardProject.kolStablecoinAddresses.push(kol);
        uint256 amount = stablecoinAmounts[i];
        rewardProject.kolStablecoinRewards[kol] = amount;
        rewardProject.kolStablecoinIndexOf[kol] = i;
        totalAmount += amount;
        emit StablecoinAllocated(rewardProjectId, kol, amount);
    }
    require(
        totalStablecoinAllocation == totalAmount,
        Amounts_Do_Not_Add_Up_To_Total_Allocation()
    );
>>  rewardProject.stablecoin.safeTransferFrom(msg.sender, address(this), totalStablecoinAllocation);
}
```

If the chosen `rewardProject.stablecoin` is fee-on-transfer (or otherwise non-standard and sends less than `totalStablecoinAllocation` to the contract), the recorded allocations (`kolStablecoinRewards`) will sum to more than the contract’s real balance, so later per-KOL `safeTransfer` calls can revert or end up effectively over-allocating relative to actual backing.

### Impact

For instance, USDT is a FOT token and they don't implement the fees. if they would, the accounting will be broken.

### Mitigation

Use before/after balance queries and record the amounts whenevery a transfer occurs.
  