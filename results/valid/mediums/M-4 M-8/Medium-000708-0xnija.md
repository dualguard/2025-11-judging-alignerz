# [000708] Off-by-One in Dividend Loop Skips Latest TVS NFT
  
  
## summary
Due to an off-by-one iteration bug in `A26ZDividendDistributor`, the highest minted TVS NFT ID is never considered when computing totals or allocating dividends. As a result, the rightful share of the holder of the highest token ID is redistributed to other holders. This occurs deterministically for every dividend round.

## Finding Description
The protocol distributes stablecoin dividends to TVS NFT holders via `A26ZDividendDistributor`:
- Contract: https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol

TVS NFTs are minted using `ERC721A`. In this implementation, token IDs start at `1`:
- `protocol/src/contracts/nft/ERC721A.sol:105–107`
`` 
function _startTokenId() internal pure virtual returns (uint256) {
    return 1;
}
 ``

The internal minted count `_totalMinted()` returns the number of tokens minted, not the highest token ID:
- `protocol/src/contracts/nft/ERC721A.sol:123–129`

`AlignerzNFT` exposes this via `getTotalMinted()`:
- `protocol/src/contracts/nft/AlignerzNFT.sol:146–148`

After minting `N` NFTs, valid token IDs are `1..N`, while `getTotalMinted()` returns `N`.

The distributor uses `nft.getTotalMinted()` to drive iteration in `getTotalUnclaimedAmounts()` and `_setDividends()`, but both loops begin at `0` and stop at `< N`. This results in iterating `i = 0..N-1`, thereby skipping `i = N` (the highest token ID).

affected function code
```solidity
function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
    uint256 len = nft.getTotalMinted();
    for (uint i; i < len;) {
        (, bool isOwned) = safeOwnerOf(i);
        if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
        unchecked { ++i; }
    }
}
 ```
- `protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:127–136`
- Link: https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L127-L136

affected function code
```solidity
function _setDividends() internal {
    uint256 len = nft.getTotalMinted();
    for (uint i; i < len;) {
        (address owner, bool isOwned) = safeOwnerOf(i);
        if (isOwned)
            dividendsOf[owner].amount += (
                unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts
            );
        unchecked { ++i; }
    }
    emit dividendsSet();
}
```
- `protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:214–223`
- Link: https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L214-L224

Key observations:
- `safeOwnerOf()` wraps `extOwnerOf()` and returns `(address(0), false)` if the call reverts (nonexistent token).
  - `protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:165–173`
- `extOwnerOf(0)` always reverts because `ERC721A._ownershipOf()` requires `tokenId >= _startTokenId()`; thus `safeOwnerOf(0)` is always “not owned”.
  - `protocol/src/contracts/nft/ERC721A.sol:180–205`, `217–219`

Root cause:
- Loops use `len = nft.getTotalMinted()` and iterate `i = 0..N-1`. Valid IDs are `1..N`, so `0` is invalid, and `N` is never visited.

Highest-impact scenario:
- When only one NFT is minted (`N = 1`), the loop visits `i = 0` only, skipping the sole valid token ID `1`. The only holder receives no dividends.

## Attack Path
- Preconditions:
  - At least one TVS NFT minted; the distributor holds stablecoin reserves for distribution.
- Steps:
  - Owner funds `A26ZDividendDistributor` with stablecoin and calls `setUpTheDividends()`.
  - `getTotalUnclaimedAmounts()` iterates `i = 0..N-1`:
    - `i = 0` is skipped since `safeOwnerOf(0)` reports nonexistence.
    - IDs `1..N-1` are included in the sum; `N` is excluded.
  - `_setDividends()` then allocates the entire `stablecoinAmountToDistribute` proportionally over `unclaimedAmountsIn[1..N-1]`, ignoring `unclaimedAmountsIn[N]`.
- Highest-impact scenario:
  - Single NFT minted (`N = 1`): the sole holder of token `1` never receives any dividends for the distribution round.

## Impact
Medium — misallocation of dividend funds causes the highest-ID holder’s share to be redistributed to others; funds are indirectly at risk via systematic accounting error.
## poc 
Run Commands

- Executed:
  - forge clean && forge build && forge test --mt test_DividendDistributor_SkipsHighestTokenId -vvv
import 
```diff
+ import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
+ import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
+ import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
```
```solidity
contract MockVestingForDividends is IAlignerzVesting {
    mapping(uint256 => Allocation) internal _allocations;

    function setAllocation(
        uint256 nftId,
        address token_,
        uint256 amount,
        uint256 claimedSeconds,
        uint256 vestingPeriod
    ) external {
        Allocation storage a = _allocations[nftId];
        a.amounts.push(amount);
        a.vestingPeriods.push(vestingPeriod);
        a.vestingStartTimes.push(block.timestamp);
        a.claimedSeconds.push(claimedSeconds);
        a.claimedFlows.push(false);
        a.token = IERC20(token_);
        a.assignedPoolId = 0;
        a.isClaimed = false;
    }

    function allocationOf(uint256 nftId) external view returns (Allocation memory) {
        return _allocations[nftId];
    }
}
```
## Recommendation
Iterate over the actual valid token ID range used by `ERC721A` (`1..N` inclusive), not `0..N-1`. Use `nft.getTotalMinted()` as the count `N`, then loop from `1` to `N`.

  