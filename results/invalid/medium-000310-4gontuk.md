# [000310] Dividend distributor over-credits total dividends after `setToken()` due to stale cached unclaimed amounts
  
  ## Finding Description
The protocol uses `A26ZDividendDistributor` to apportion a `stablecoin` pool among TVS NFT holders based on each NFT’s “unclaimed token value”.

Key state in [`A26ZDividendDistributor`](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L214-L224):

- `mapping(uint256 => uint256) unclaimedAmountsIn;` — cached per‑NFT “unclaimed amount”.
- `uint256 public totalUnclaimedAmounts;` — sum of unclaimed amounts used as denominator in distribution.
- `uint256 public stablecoinAmountToDistribute;` — amount of stablecoin to be spread over all NFTs.
- `mapping(address => Dividend) dividendsOf;` — final per-user dividend entitlements.

Core functions:

- [`setUpTheDividends()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L111-L114):
  - Calls `_setAmounts()` then `_setDividends()`.
- [`_setAmounts()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L207-L211):
  - Sets `stablecoinAmountToDistribute = stablecoin.balanceOf(address(this));`
  - Sets `totalUnclaimedAmounts = getTotalUnclaimedAmounts();`
- `getTotalUnclaimedAmounts()`:
  - Iterates `nft.getTotalMinted()` and for each owned NFT calls `getUnclaimedAmounts(nftId)` and **sums the return values** into a local `_totalUnclaimedAmounts`, which is then assigned to `totalUnclaimedAmounts`.
- [`getUnclaimedAmounts(uint256 nftId)`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161):
  - Reads allocation data from `vesting.allocationOf(nftId)` and computes the total unclaimed amount across all flows.
  - **Crucially**:

    ```solidity
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;

        uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
        uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
        uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
        bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
        uint256 len = vesting.allocationOf(nftId).amounts.length;
        for (uint i; i < len;) {
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

  - The early return on `address(token) == allocation.token` **does not touch `unclaimedAmountsIn[nftId]` at all**; the cached value remains whatever it was from previous calls (if any).
- [`_setDividends()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L214-L224):
  - Iterates NFTs again and uses **the cached mapping**:

    ```solidity
    function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len;) {
            (address owner, bool isOwned) = safeOwnerOf(i);
            if (isOwned) dividendsOf[owner].amount += (
                unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts
            );
            unchecked {
                ++i;
            }
        }
        emit dividendsSet();
    }
    ```

The interaction with [`setToken()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L91-L94) is problematic:

- `setToken(address _token)` simply updates the `token` reference:

  ```solidity
  function setToken(address _token) external onlyOwner {
      token = IERC20(_token);
      emit tokenSet(_token);
  }
  ```

- After changing `token`, the condition `address(token) == address(vesting.allocationOf(nftId).token)` may newly become true for some NFTs that were previously processed normally.

**Root cause**

1. In an initial configuration where `address(token) != allocation.token` for all NFTs, `getUnclaimedAmounts()` computes a positive `amount` for each NFT and sets `unclaimedAmountsIn[nftId] = amount`. `totalUnclaimedAmounts` is correctly computed as the sum of these per‑NFT values, and `_setDividends()` distributes `stablecoinAmountToDistribute` proportionally. At this point, cache and `totalUnclaimedAmounts` are consistent.

2. Later, the owner calls `setToken()` so that for some NFT(s) `address(token) == allocation.token` becomes true:
   - On the next `_setAmounts()` / `getTotalUnclaimedAmounts()`:
     - `getUnclaimedAmounts()` for those NFTs **returns `0` immediately** and **does not update `unclaimedAmountsIn[nftId]`**, which retains its old non-zero value from a previous round.
     - `totalUnclaimedAmounts` is recomputed **without** including any contribution from these NFTs (they return `0` and are effectively excluded from the denominator).

3. `_setDividends()` then runs using:
   - `unclaimedAmountsIn[nftId]` for **all** NFTs (including those now skipped by `getUnclaimedAmounts()` due to the early return).
   - `totalUnclaimedAmounts` that only sums unclaimed values from NFTs *not* short-circuited by the early return.

This leads to a mismatch:

- Some NFTs have **stale, positive** `unclaimedAmountsIn[nftId]` not reflected in `totalUnclaimedAmounts`.
- `_setDividends()` multiplies `unclaimedAmountsIn[nftId]` by `stablecoinAmountToDistribute / totalUnclaimedAmounts` for all NFTs, including those whose `unclaimedAmountsIn[nftId]` no longer contributed to `totalUnclaimedAmounts`.

In other words:

- Let `U_stale = Σ (stale cached unclaimedAmountsIn[i] over NFTs newly excluded by `address(token) == allocation.token`)`.
- Let `U_fresh = Σ (fresh unclaimed amounts for NFTs still included) = totalUnclaimedAmounts`.

The newly added dividends in a distribution round become:

- `Σ_i (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts)`
- `= (U_fresh + U_stale) * stablecoinAmountToDistribute / U_fresh`
- which is **strictly greater than** `stablecoinAmountToDistribute` whenever `U_stale > 0`.

Thus, after a `setToken()` reconfiguration, the contract can over-credit `dividendsOf[owner].amount` across all holders in a given distribution round compared with the configured `stablecoinAmountToDistribute`.


## Impact
Medium. The bug does not make the contract send more tokens than it holds, but it causes systematic mis-accounting of dividends after `setToken()` and can lead to unfair over-allocation to some holders and potential permanent inability for others to claim their expected share once the pool is exhausted.

## Likelihood
Medium. The issue requires a specific sequence (distributor configured, `setUpTheDividends()` used, then `setToken()` changed and dividend setup repeated), which is realistic under active treasury management or token migration scenarios

## Proof of Concept
Add the PoC to `protocol/test/AlignerzVestingProtocolTest.t.sol`

1. Import these

```solidity
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
import {IAlignerzNFT} from "../src/interfaces/IAlignerzNFT.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
```

2. Add the Harness contract

```solidity
contract A26ZDividendDistributorHarness is A26ZDividendDistributor {
    constructor(
        address _vesting,
        address _nft,
        address _stablecoin,
        uint256 _startTime,
        uint256 _vestingPeriod,
        address _token
    ) A26ZDividendDistributor(_vesting, _nft, _stablecoin, _startTime, _vestingPeriod, _token) {}

    function getUnclaimedAmountIn(uint256 nftId) external view returns (uint256) {
        return unclaimedAmountsIn[nftId];
    }

    function getDividendData(address user) external view returns (uint256 amount, uint256 claimedSeconds) {
        Dividend memory d = dividendsOf[user];
        return (d.amount, d.claimedSeconds);
    }
}
```

3. Add these Mocks

```solidity
contract MockDividendNFT is IAlignerzNFT {
    uint256 public totalMinted;
    mapping(uint256 => address) internal owners;

    function mint(address to) external override returns (uint256) {
        uint256 tokenId = totalMinted;
        owners[tokenId] = to;
        totalMinted++;
        return tokenId;
    }

    function extOwnerOf(uint256 tokenId) external view override returns (address) {
        address owner = owners[tokenId];
        require(owner != address(0), "NFT: nonexistent");
        return owner;
    }

    function burn(uint256 tokenId) external override {
        owners[tokenId] = address(0);
    }

    function getTotalMinted() external view override returns (uint256) {
        return totalMinted;
    }
}

contract MockVestingForDividends is IAlignerzVesting {
    mapping(uint256 => Allocation) internal _allocations;

    function allocationOf(uint256 nftId) external view override returns (Allocation memory) {
        return _allocations[nftId];
    }

    function setSingleAllocation(
        uint256 nftId,
        uint256 amount,
        uint256 vestingPeriod,
        uint256 claimedSeconds,
        bool claimedFlow,
        IERC20 token_
    ) external {
        uint256[] memory amounts = new uint256[](1);
        uint256[] memory vestingPeriods = new uint256[](1);
        uint256[] memory vestingStartTimes = new uint256[](1);
        uint256[] memory claimedSecondsArr = new uint256[](1);
        bool[] memory claimedFlows = new bool[](1);

        amounts[0] = amount;
        vestingPeriods[0] = vestingPeriod;
        vestingStartTimes[0] = 0;
        claimedSecondsArr[0] = claimedSeconds;
        claimedFlows[0] = claimedFlow;

        Allocation storage alloc = _allocations[nftId];
        alloc.amounts = amounts;
        alloc.vestingPeriods = vestingPeriods;
        alloc.vestingStartTimes = vestingStartTimes;
        alloc.claimedSeconds = claimedSecondsArr;
        alloc.claimedFlows = claimedFlows;
        alloc.isClaimed = false;
        alloc.token = token_;
        alloc.assignedPoolId = 0;
    }
}
```

4. Add the PoC test to `AlignerzVestingProtocolTest`

```solidity
    function test_DividendDistributor_StaleUnclaimedAmountsAfterSetToken() public {
        // Use separate mock contracts to focus on dividend logic
        MockDividendNFT mockNft = new MockDividendNFT();
        MockVestingForDividends mockVesting = new MockVestingForDividends();

        // Two different TVS tokens for allocations
        Aligners26 tvsTokenA = token; // from setUp
        Aligners26 tvsTokenB = new Aligners26("SecondTVS", "TVS2");
        Aligners26 initialToken = new Aligners26("InitialTVS", "INIT");

        // Deploy dividend distributor with a token different from both allocation tokens
        A26ZDividendDistributorHarness distributor = new A26ZDividendDistributorHarness(
            address(mockVesting),
            address(mockNft),
            address(usdt),
            block.timestamp,
            365 days,
            address(initialToken)
        );

        // Mint two NFTs to two holders (IDs 0 and 1)
        address holder0 = address(0x100);
        address holder1 = address(0x200);
        uint256 nftId0 = mockNft.mint(holder0);
        uint256 nftId1 = mockNft.mint(holder1);
        assertEq(nftId0, 0);
        assertEq(nftId1, 1);

        // Set allocations with claimedSeconds > 0 to avoid the known infinite loop bug
        // NFT 0 uses tvsTokenA, NFT 1 uses tvsTokenB
        uint256 amount0 = 100e6;
        uint256 amount1 = 200e6;
        uint256 vestingPeriod = 100;
        uint256 claimedSeconds = 50;

        mockVesting.setSingleAllocation(
            nftId0,
            amount0,
            vestingPeriod,
            claimedSeconds,
            false,
            IERC20(address(tvsTokenA))
        );

        mockVesting.setSingleAllocation(
            nftId1,
            amount1,
            vestingPeriod,
            claimedSeconds,
            false,
            IERC20(address(tvsTokenB))
        );

        // Fund distributor with stablecoin once
        uint256 deposit = 1_000e6;
        usdt.transfer(address(distributor), deposit);

        // ROUND 1: populate unclaimedAmountsIn for both NFTs
        distributor.setUpTheDividends();

        uint256 u0Round1 = distributor.getUnclaimedAmountIn(nftId0);
        uint256 u1Round1 = distributor.getUnclaimedAmountIn(nftId1);
        uint256 totalUnclaimedRound1 = distributor.totalUnclaimedAmounts();
        assertGt(u0Round1, 0);
        assertGt(u1Round1, 0);
        assertEq(totalUnclaimedRound1, u0Round1 + u1Round1);

        (uint256 d0Before,) = distributor.getDividendData(holder0);
        (uint256 d1Before,) = distributor.getDividendData(holder1);

        // ROUND 2: change token so that NFT 0 is now excluded by the early return,
        // leaving its unclaimedAmountsIn entry stale, while NFT 1 is still included.
        distributor.setToken(address(tvsTokenA));

        distributor.setAmounts();
        uint256 totalUnclaimedRound2 = distributor.totalUnclaimedAmounts();
        uint256 u0Round2 = distributor.getUnclaimedAmountIn(nftId0);
        uint256 u1Round2 = distributor.getUnclaimedAmountIn(nftId1);

        // NFT 0's cached value stays stale, but it is no longer counted in totalUnclaimedAmounts
        assertEq(u0Round2, u0Round1);
        assertEq(u1Round2, u1Round1);
        assertEq(totalUnclaimedRound2, u1Round2);

        // Set dividends for the second round using the mismatched state
        distributor.setDividends();

        (uint256 d0After,) = distributor.getDividendData(holder0);
        (uint256 d1After,) = distributor.getDividendData(holder1);

        uint256 inc0 = d0After - d0Before;
        uint256 inc1 = d1After - d1Before;
        uint256 totalInc = inc0 + inc1;

        uint256 stablecoinToDistributeRound2 = distributor.stablecoinAmountToDistribute();

        // Because NFT 0's stale unclaimedAmountsIn is not in the denominator but is
        // still used as a numerator in _setDividends, the total newly credited dividends
        // for round 2 exceed the configured stablecoinAmountToDistribute.
        assertGt(totalInc, stablecoinToDistributeRound2);
    }
```


High-level PoC structure:

- Deploys small harness/mocks to isolate the dividend logic:
  - `A26ZDividendDistributorHarness` extends `A26ZDividendDistributor` and exposes:
    - `getUnclaimedAmountIn(uint256 nftId)` to read `unclaimedAmountsIn[nftId]`.
    - `getDividendData(address user)` to read `dividendsOf[user]`.
  - `MockDividendNFT` implements `IAlignerzNFT` with:
    - Simple `mint()`, `extOwnerOf()`, `burn()`, `getTotalMinted()`.
  - `MockVestingForDividends` implements `IAlignerzVesting` with:
    - `setSingleAllocation(...)` to set a 1-flow `Allocation` per NFT.
    - `allocationOf(nftId)` returning that struct.

- The test:
  1. Deploys the mocks and harness with an initial `token = INIT` that differs from both allocation tokens.
  2. Mints two NFTs (`0` and `1`) to two holders.
  3. Sets allocations for both NFTs with:
     - Positive `amount`s and `claimedSeconds > 0` to avoid the known loop bug in `getUnclaimedAmounts()`.
  4. Transfers a fixed amount of `MockUSD` (`deposit`) to the distributor.
  5. Calls `setUpTheDividends()` once:
     - Verifies that `unclaimedAmountsIn[0]` and `unclaimedAmountsIn[1]` are non-zero.
     - Verifies that `totalUnclaimedAmounts == unclaimedAmountsIn[0] + unclaimedAmountsIn[1]`.
     - Records initial `dividendsOf[holder0].amount` and `dividendsOf[holder1].amount`.
  6. Calls `setToken(address(tvsTokenA))` so that NFT 0 now matches `token` and will be short-circuited in `getUnclaimedAmounts()`.
  7. Calls `setAmounts()` again:
     - Confirms `totalUnclaimedAmounts` now equals only the unclaimed amount of NFT 1.
     - Confirms `unclaimedAmountsIn[0]` is unchanged (stale), while `unclaimedAmountsIn[1]` is still the expected value.
  8. Calls `setDividends()`:
     - Computes increments `inc0` and `inc1` to each holder’s `dividendsOf` based on the second round.
     - Asserts that `inc0 + inc1` (the sum of new dividends credited in this round) is **greater than** `stablecoinAmountToDistribute`.

This assertion passing confirms that after a `setToken()` change, the combination of stale `unclaimedAmountsIn` and a recomputed `totalUnclaimedAmounts` yields over-credited dividends for the same configured pool.

To run the PoC:

- From the `protocol` directory , execute:

  ```bash
  forge clean
  forge build
  forge test --match-test test_DividendDistributor_StaleUnclaimedAmountsAfterSetToken 
  ```

- The test should pass and report `test_DividendDistributor_StaleUnclaimedAmountsAfterSetToken()` as successful, with the key `assertGt(totalInc, stablecoinToDistributeRound2)` indicating the over-crediting.

## Recommendation
Ensure that the cached `unclaimedAmountsIn[nftId]` is always kept consistent with the values used to compute `totalUnclaimedAmounts`, especially in the presence of configuration changes like `setToken()`.

  