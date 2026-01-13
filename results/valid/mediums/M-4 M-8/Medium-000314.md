# [000314] Highest-ID TVS NFT holder never receives dividends
  
  ## Finding Description

The protocol introduces an [`A26ZDividendDistributor`](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol) contract to distribute stablecoin dividends to holders of TVS NFTs (`AlignerzNFT`). TVS NFTs are minted by `AlignerzVesting` via the `AlignerzNFT` contract, which is built on top of `ERC721A`.

In `ERC721A`, token IDs start at `1` and increase sequentially:

[ERC721A.sol#L105-L107](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/ERC721A.sol#L105-L107)
```solidity
function _startTokenId() internal pure virtual returns (uint256) {
    return 1;
}
```

The internal counter `_currentIndex` is initialized to `_startTokenId()` in the `ERC721A` constructor, and `_totalMinted()` returns the **number of tokens minted**, not the highest token ID:

```solidity
constructor(string memory name_, string memory symbol_) {
    _name = name_;
    _symbol = symbol_;
    _currentIndex = _startTokenId();
}

function _totalMinted() internal view returns (uint256) {
    unchecked {
        return _currentIndex - _startTokenId();
    }
}
```

`AlignerzNFT` exposes this as:

```solidity
function getTotalMinted() external view returns (uint256 totalMinted){
    totalMinted = _totalMinted();
}
```

Consequently, after minting `N` NFTs:

- Valid token IDs are `1, 2, ..., N`.
- `getTotalMinted()` returns `N` (the **count** of minted NFTs, not the max ID).

### How the dividend distributor iterates over NFTs

`A26ZDividendDistributor` uses `nft.getTotalMinted()` to determine how many NFT IDs to iterate over in [`getTotalUnclaimedAmounts()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L127-L136) and [`_setDividends()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L214-L224) :

```solidity
function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
    uint256 len = nft.getTotalMinted();
    for (uint i; i < len;) {
        (, bool isOwned) = safeOwnerOf(i);
        if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
        unchecked {
            ++i;
        }
    }
}
```

```solidity
function _setDividends() internal {
    uint256 len = nft.getTotalMinted();
    for (uint i; i < len;) {
        (address owner, bool isOwned) = safeOwnerOf(i);
        if (isOwned)
            dividendsOf[owner].amount += (
                unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts
            );
        unchecked {
            ++i;
        }
    }
    emit dividendsSet();
}
```

Key points:

- `len = nft.getTotalMinted()` is `N`, the number of minted NFTs.
- Both loops start from `i = 0` and run while `i < len`, so they visit `i = 0, 1, ..., N - 1`.

However, valid token IDs are `1..N`:

- **Index `0`**:  
  - `safeOwnerOf(0)` wraps `nft.extOwnerOf(0)` and catches the revert from `_ownershipOf(0)`, returning `(address(0), false)`.  
  - This is a non-existent token and is always skipped; the iteration does nothing.

- **Indices `1..N-1`**:  
  - These correspond to real token IDs and are processed normally.

- **Index `N` (the highest minted ID)**:  
  - The loops never reach `i = N` because the condition is `i < len` with `len = N`.  
  - As a result, the NFT with token ID `N` is **completely ignored** in:
    - `getTotalUnclaimedAmounts()` (its unclaimed TVS value is not included in the denominator), and
    - `_setDividends()` (no dividends are credited based on `unclaimedAmountsIn[N]`).

### Concrete scenarios and highest-impact case

1. **Single NFT minted (N = 1)**

   - Valid token IDs: `[1]`.
   - `getTotalMinted()` returns `1`.
   - Loops run with `i = 0` only:
     - `safeOwnerOf(0)` always returns `(0, false)`, so `getUnclaimedAmounts(1)` is never called.
     - `unclaimedAmountsIn[1]` is not populated, and `_setDividends()` never allocates dividends to the owner of token `1`.

   **Effect:** the only NFT in existence never receives any dividends, even if it has a valid non-zero TVS allocation.

2. **Multiple NFTs minted (N > 1)**

   - Valid token IDs: `1, 2, ..., N`.
   - Loops cover `i = 0, 1, ..., N-1`:
     - `0` is always skipped as non-existent.
     - Only `1..N-1` are included in the unclaimed-amount sum and in dividend allocation.
     - Token ID `N` (the highest minted NFT) is **never processed** in either step.

   **Effect:** the highest-ID NFT holder is systematically excluded from all dividend calculations.

3. **Economic impact**

   - `getTotalUnclaimedAmounts()` sums only unclaimed amounts for IDs `1..N-1`.
   - `_setDividends()` distributes the **entire** `stablecoinAmountToDistribute` over `unclaimedAmountsIn[1..N-1]`, ignoring `unclaimedAmountsIn[N]`.
   - The rightful share of the holder of token ID `N` is effectively redistributed to the other holders.

## Impact
Medium. Funds are indirectly at risk because dividend stablecoin allocations are systematically mis-accounted: the highest-ID TVS NFT holder never receives dividends for a given distribution, and their share is redistributed to other holders.

## Likelihood

High. The bug is deterministic and always present whenever `A26ZDividendDistributor` is used: as soon as at least one NFT is minted and dividends are set up, the highest token ID is guaranteed to be skipped, without any special preconditions or adversarial actions.

## Proof of Concept

Add the PoC to `protocol/test/AlignerzVestingProtocolTest.t.sol` :

1. Import these first:

```solidity
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
```

2. Add this helper Mock:
`MockVestingForDividends` implementing `IAlignerzVesting`’s `allocationOf()` with an internal mapping of `Allocation`s.

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


3. PoC test:
`test_DividendDistributor_SkipsHighestTokenId()` appended to the end of `AlignerzVestingProtocolTest`.

```solidity
    function test_DividendDistributor_SkipsHighestTokenId() public {
        // Setup: use existing NFT and tokens, but a dedicated mock vesting and distributor
        MockVestingForDividends mockVesting = new MockVestingForDividends();

        // Mint two NFTs so that token IDs are 1 and 2 and total minted is 2
        address holder1 = makeAddr("holder1");
        address holder2 = makeAddr("holder2");

        uint256 id1 = nft.mint(holder1);
        uint256 id2 = nft.mint(holder2);

        assertEq(id1, 1);
        assertEq(id2, 2);

        uint256 totalMinted = nft.getTotalMinted();
        assertEq(totalMinted, 2);

        // Configure mock vesting so both NFTs have identical unclaimed amounts
        Aligners26 otherToken = new Aligners26("OtherToken", "OTH");
        uint256 flowAmount = 100 ether;
        uint256 vestingPeriod = 100;
        uint256 claimedSeconds = 50; // non-zero to avoid the getUnclaimedAmounts() infinite loop

        mockVesting.setAllocation(id1, address(otherToken), flowAmount, claimedSeconds, vestingPeriod);
        mockVesting.setAllocation(id2, address(otherToken), flowAmount, claimedSeconds, vestingPeriod);

        // Deploy the dividend distributor with:
        // - mock vesting
        // - real NFT contract (with token IDs 1 and 2)
        // - USDT as the stablecoin to distribute
        // - Aligners26 token as the configured TVS token (different from otherToken)
        uint256 startTime = block.timestamp;
        uint256 divVestingPeriod = 30 days;

        A26ZDividendDistributor distributor = new A26ZDividendDistributor(
            address(mockVesting),
            address(nft),
            address(usdt),
            startTime,
            divVestingPeriod,
            address(token)
        );

        // Fund the distributor with stablecoin for dividends
        uint256 dividendAmount = 1_000 ether;
        usdt.mint(address(distributor), dividendAmount);

        // Owner (this test contract) sets up the dividends
        distributor.setUpTheDividends();

        // Fast-forward beyond the dividend vesting period so all configured dividends are claimable
        vm.warp(startTime + divVestingPeriod + 1);

        uint256 holder1Before = usdt.balanceOf(holder1);
        uint256 holder2Before = usdt.balanceOf(holder2);

        // Holder 1 (tokenId 1) claims dividends
        vm.prank(holder1);
        distributor.claimDividends();

        // Holder 2 (tokenId 2, the highest tokenId) also attempts to claim dividends
        vm.prank(holder2);
        distributor.claimDividends();

        uint256 holder1After = usdt.balanceOf(holder1);
        uint256 holder2After = usdt.balanceOf(holder2);

        // Due to the off-by-one in getTotalUnclaimedAmounts/setDividends, only tokenId 1 is processed
        assertGt(holder1After - holder1Before, 0, "holder1 should receive dividends");
        assertEq(holder2After - holder2Before, 0, "holder2 (highest tokenId) is incorrectly excluded from dividends");
    }
```

This clearly demonstrates that only non-highest token IDs are credited, even though both NFTs have identical unclaimed TVS allocations.

### How to run the PoC

From the `protocol` directory:

```bash
forge clean
forge build
forge test --mt test_DividendDistributor_SkipsHighestTokenId -vvv
```

## Recommendation

The loops in `getTotalUnclaimedAmounts()` and `_setDividends()` should iterate over the **actual token ID range** used by `AlignerzNFT` (starting from 1) rather than assuming `0..getTotalMinted()-1` are valid IDs. The simplest practical fix, given the current contracts, is:

- Use `nft.getTotalMinted()` as the **count** `N`.
- Iterate token IDs from `1` to `N` inclusive.


  