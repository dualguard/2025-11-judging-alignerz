# [000127] Dividend entitlements remain bound to original holder address, not current NFT owner
  
  ### Summary

The A26ZDividendDistributor contract binds dividend entitlements to the address that owned NFTs at the moment `_setDividends()` is called and never rebinds these entitlements when NFTs are later transferred. `claimDividends()` then pays dividends based solely on `msg.sender`, without checking current NFT ownership. 

### Root Cause

Dividend state is tracked per address:

```solidity
/// @notice stablecoin balances allocated to TVS holders (dividends)
mapping(address => Dividend) dividendsOf;

struct Dividend {
    uint256 amount;         // amount in USD in 1e6
    uint256 claimedSeconds; // seconds claimed
}
```

During setup, `_setDividends()` iterates over all minted NFT IDs, fetches the snapshot owner for each ID, and permanently credits their dividend principal:

```solidity
function _setDividends() internal {
    uint256 len = nft.getTotalMinted();
    for (uint256 i; i < len;) {
        (address owner, bool isOwned) = safeOwnerOf(i);
        if (isOwned) {
            dividendsOf[owner].amount +=
                (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
        }
        unchecked {
            ++i;
        }
    }
    emit dividendsSet();
}
```

Here, `owner` is the address that owns NFT `i` at the time `_setDividends()` runs. The resulting `dividendsOf[owner].amount` is not keyed by NFT ID or any epoch; it is simply accumulated on the address.

The claim path in `claimDividends()` looks up entitlements only by `msg.sender`:

```solidity
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

`claimDividends()` does not:

- Re-check the caller’s current NFT holdings via `safeOwnerOf()` or any balance function.
- Migrate or adjust `dividendsOf` entries when NFTs are transferred.
- Distinguish between historical and current owners for a given NFT.

The NFT contract `AlignerzNFT` is a standard ERC721A-based contract and does not call into `A26ZDividendDistributor` on transfer or burn. There is no on-transfer hook that would reassign or adjust dividend rights.

As a result, once `_setDividends()` is executed, the addresses that held NFTs at that snapshot time retain their dividend principal in `dividendsOf`, even if they later transfer all NFTs. Subsequent buyers of those NFTs do not automatically receive any of the already-assigned dividends, and have no direct way to claim them through `A26ZDividendDistributor`.


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. At time T0, `_setDividends()` is called.
   - NFT #1 is owned by Alice.
   - `_setDividends()` calls `safeOwnerOf(1)` and credits `dividendsOf[Alice].amount` with her share of `stablecoinAmountToDistribute`.
2. At time T1, Alice transfers NFT #1 to Bob through the standard ERC721A transfer (no interaction with the dividend contract).
3. At time T2, `claimDividends()` is called:
   - Alice calls `claimDividends()` and receives dividends based on `dividendsOf[Alice]`, even though she no longer owns NFT #1.
   - Bob calls `claimDividends()` and receives nothing because `dividendsOf[Bob]` was never credited, despite being the current NFT owner.
4. This sequence can be repeated for multiple NFTs and multiple distribution rounds, with historical holders retaining claim rights and buyers lacking them.

### Impact

Severity is High because this causes loss of yield, as this allows past holders to retain dividend streams after selling their NFTs and preventing new holders from claiming already-assigned dividends.

### PoC

_No response_

### Mitigation

_No response_
  