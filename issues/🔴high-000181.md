# [000181] `allocationOf` mapping is not updated in `mergeTVS` function, leading to loss of rewards in A26ZDividendDistributor.
  
  ### Summary

`allocationOf` mapping keeps track of allocations given a NFT ID.  It is updated in `distributeRemainingRewardTVS` in the context of reward projects. It is also updated in `claimNFT` function for bidding projects.

Contrary to `splitTVS` function which effectively updated `allocationOf` in the loop:

```solidity
allocationOf[nftId] = newAlloc;
```

The `mergeTVS` function doesn't update the `allocationOf ` mapping while it should.


### Root Cause

There are 2 places in `mergeTVS` function where `allocationOf` mapping should be updated:

1. https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1023

After the `for` loop, the storage `mergedTVS` struct has been updated and includes the flows of the merged TVS. Hence, `allocationOf[mergedNftId]` should be updated. This missing update has severe consequences.

2. https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1047

In `_merge` internal function, once the NFT to be merged has been merged, it is burnt. But `allocationOf[nftId]` should be deleted. This missing update has no real consequence.

### Attack Path

There is no specific attack path. Every time a merge happens, the `allocationOf` mapping is not updated for both the resulting NFT and the burnt NFTs. This has consequences in the _A26ZDividendDistributor_ contract.

### Impact

The impact of this issue is actually high. Indeed,  it results in `allocationOf` public storage variable being incorrect, not reflecting the actual state. The getter function associated to this storage variable is used by the _A26ZDividendDistributor_ contract in `getUnclaimedAmounts` function. This function is used when setting amounts for a round of dividend distribution. 

```solidity
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
        uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
        uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
        uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
        bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
        uint256 len = vesting.allocationOf(nftId).amounts.length;
        for (uint256 i; i < len;) {
            if (claimedFlows[i]) continue;
            if (claimedSeconds[i] == 0) {
                amount += amounts[i];
                continue;
            }
            uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
            uint256 unclaimedAmount = amounts[i] - claimedAmount;
            amount += unclaimedAmount;
            unchecked {
                ++i;
            }
        }
        unclaimedAmountsIn[nftId] = amount;
    }
```

Because the allocation of the NFT that results from the merge has an amount lower than what it should be, it means that the line : 
```solidity
unclaimedAmountsIn[nftId] = amount
```
will lead to loss of rewards for NFT holder. Indeed, `amount` is deflated and only reflects the first flow of the NFT that necessarily contains multiple flows after a merge.

Also, in `getTotalUnclaimedAmounts` function, the returning value `_totalUnclaimedAmounts` will be less than it should, given that `getUnclaimedAmounts(i)` will return less than it should for the merged NFT.

```solidity
    function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
        // @audit why not directly use nft.totalSupply()?
        uint256 len = nft.getTotalMinted();
        for (uint256 i; i < len;) {
            (, bool isOwned) = safeOwnerOf(i);
            if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
            unchecked {
                ++i;
            }
        }
```
This means that `_setDividends` function will wrongly set dividends for all users to an higher value than expected:

```solidity
    function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
        for (uint256 i; i < len;) {
            (address owner, bool isOwned) = safeOwnerOf(i);
            if (isOwned) {
                dividendsOf[owner].amount += (unclaimedAmountsIn[i]
                        * stablecoinAmountToDistribute
                        / totalUnclaimedAmounts); // @audit we divide by an underestimated value
            }
            unchecked {
                ++i;
            }
        }
        emit dividendsSet();
    }
```

But ultimately, the main problem is that dividends of users who merged NFTs will be less, leading to loss of rewards for them.

### Mitigation

Make sure to update correctly `allocationOf` mapping in `mergeTVS` and `_merge` functions.
  