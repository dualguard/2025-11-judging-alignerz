# [000766] Dividends snapshotted per address, not per NFT, allowing ex‑owner to claim rewards
  
  ### Summary

Assigning dividends directly to addresses at `_setDividends()` instead of to specific NFT IDs causes later NFT transfers to decouple dividend entitlements from TVS ownership: sellers can keep an epoch’s dividends after selling their NFTs, while buyers receive no dividends for that epoch despite holding the TVS.

### Root Cause

In `A26ZDividendDistributor`, the per-epoch dividend amounts are stored in `dividendsOf[owner]` based on ownership at snapshot time, and subsequent claims never check NFT ownership again.

```solidity
// File: A26ZDividendDistributor.sol

function _setDividends() internal {
    uint256 len = nft.getTotalMinted();
    for (uint i; i < len;) {
        (address owner, bool isOwned) = safeOwnerOf(i);
        if (isOwned)
            dividendsOf[owner].amount += (
                unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts
            );  //@audit ownership only checked at snapshot
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

The mapping key is the address, not `(nftId, epoch)` or similar, and no subsequent call to `safeOwnerOf` is made in `claimDividends`. Once `_setDividends` runs, the right to that epoch’s dividends is fixed to the address that held each NFT at snapshot time, regardless of later transfers of the NFTs.

### Internal Pre-conditions

1. The TVS NFTs represented by `nft` are tradable between arbitrary addresses.  
2. `_setAmounts` and `_setDividends` are called to initialize a dividend epoch, filling `dividendsOf[owner].amount` based on ownership at that snapshot.  
3. `claimDividends` uses only `msg.sender` and `dividendsOf[msg.sender]` to determine payments, without checking current NFT holdings.

### External Pre-conditions

1. TVS NFTs can be transferred on a secondary market after dividends have been configured for an epoch.  
2. Buyers reasonably expect that purchasing a TVS NFT before or during a dividend epoch entitles them to that epoch’s dividends tied to the NFT.

### Attack Path

An epoch is set up by the operator calling `setUpTheDividends` or equivalent, which internally runs `_setDividends` and credits `dividendsOf[owner].amount` for each address based on ownership at that moment. A seller then transfers the NFT to a buyer after `_setDividends` has run but before `claimDividends`. The dividends for that epoch remain credited to the seller’s address in `dividendsOf`, and the buyer’s address has no associated dividend balance for that epoch, even though the buyer now holds the NFT. The seller can claim the entire epoch’s dividends even after having sold the TVS, while the buyer receives zero for that epoch. Repeated trading around snapshot times can systematically extract value from uninformed buyers who expect dividends to follow the NFT.

### Impact

Dividend entitlements are bound to addresses at a single snapshot rather than to ongoing NFT ownership. Sellers keep dividends for epochs that buyers believe they are purchasing into, and buyers receive less than expected. Over time this leads to divergence between “who owns the TVS” and “who receives the dividends,” enabling ex‑dividend trading patterns that transfer dividend value from buyers to sellers without any on-chain indication, and undermining economic fairness of the dividend mechanism. Severity is medium–high from an economic and design perspective, especially in a secondary market where participants may not understand this behavior.

### POC


### Mitigation

A more robust design would tie dividends to NFT IDs and resolve ownership at claim time, or maintain per‑epoch, per‑NFT entitlements that are explicitly transferable with the NFT. 
  