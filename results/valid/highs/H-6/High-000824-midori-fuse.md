# [000824] Dividend distribution can be permanently DOSed with out-of-gas by repeatedly splitting an NFT and inflating the total minted
  
  ### Summary

An attacker can repeatedly split a single NFT into thousands of pieces, causing `getTotalUnclaimedAmounts()` to run out of gas and permanently disable the dividend distribution system.

### Root Cause


In [A26ZDividendDistributor.sol:127-136](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L127-L136), `getTotalUnclaimedAmounts()` iterates through all minted NFTs without any bound:

```solidity
function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
    uint256 len = nft.getTotalMinted(); // @audit Unbounded - grows with every NFT minted
    for (uint i; i < len;) {
        (, bool isOwned) = safeOwnerOf(i);
        if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
        // ...
    }
}
```

The total number of minted NFTs can be artificially inflated at minimal cost. In [AlignerzVesting.sol:1054-1107](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054-L1107), `splitTVS()` has no limit on how many NFTs can be created from a single split:

```solidity
function splitTVS(
    uint256 projectId,
    uint256[] calldata percentages,
    uint256 splitNftId
) external returns (uint256, uint256[] memory) {
    //...
    uint256 nbOfTVS = percentages.length;
    //...
    uint256 sumOfPercentages;
    for (uint256 i; i < nbOfTVS; ) {
        uint256 percentage = percentages[i];
        sumOfPercentages += percentage;
        //...
    }
    require(
        sumOfPercentages == BASIS_POINT,
        Percentages_Do_Not_Add_Up_To_One_Hundred()
    );
    //...
}
```

By splitting one NFT into thousands of pieces repeatedly, an attacker can artificially inflate the total minted NFT count to make `getTotalUnclaimedAmounts()` exceed gas limits.

### Internal Pre-conditions

1. Attacker owns at least one NFT from a project configured in the dividend distributor (easily obtained by bidding minimal amount with maximum vesting period)

### External Pre-conditions

None

### Attack Path

1. Attacker bids **minimal amount** with **maximum vesting period** to obtain one NFT from a project configured in the dividend distributor
    - They can also buy such an NFT on the secondary market. Any NFT will work, regardless of cost.
2. Attacker repeatedly calls `splitTVS()` to inflate the total NFT count:
   - Uses percentage array like `[10000, 0, 0, 0, ...]` where first element is 10,000 and rest are zeros
   - **Split 1**: Creates 9,999 empty NFTs (total: 10,000 NFTs)
   - **Split 2**: Creates 9,999 more empty NFTs (total: 19,999 NFTs)
   - Repeats until total minted NFT count reaches desired threshold (e.g., 100,000+ NFTs)
   - **Attack cost is minimal**—attacker keeps all tokens and only pays gas
3. Owner calls `setAmounts()` to set up dividend distribution, which will call `getTotalUnclaimedAmounts()`
4. Function iterate through 100,000+ NFTs. The loop exceeds transaction/block gas limit and the transaction reverts with an **out-of-gas** error


### Impact

Attacker can break a core protocol feature and prevent all users from receiving dividends with minimal cost.


### PoC

_No response_

### Mitigation

Any loop that relies on `nft.getTotalMinted()` should not be used. This is because NFT minting is effectively permissionless, even A26Z TVS NFTs can be splitted permissionlessly.

The suggested mitigation is to allow the admin to specify which NFTs should be eligible for diviend distribution.

  