# [000323] A26Z TVS holders never receive dividends
  
  

## Finding Description
The protocol introduces an `A26ZDividendDistributor` contract intended to distribute stablecoin dividends proportionally to holders of Token Vesting Schedules (TVSs) whose underlying token is A26Z. These TVSs are represented by NFTs created and managed by the `AlignerzVesting` contract, which exposes per-NFT allocations via the `allocationOf()` getter, returning an `Allocation` struct that includes the TVS token and vesting amounts.

In `AlignerzVesting`, all TVSs for a given project are explicitly parameterized by a token field that is set to the project’s TVS token (for example, A26Z):

- `RewardProject.token` / `BiddingProject.token` hold the TVS ERC‑20 for that project.
- When a TVS NFT is minted or distributed (e.g. via reward projects or bidding projects), the corresponding `Allocation` stored in `allocationOf[nftId]` has its `token` field set to the project TVS token. Thus, for an A26Z project, `allocationOf(nftId).token` is A26Z for all associated NFTs.

In [`A26ZDividendDistributor`](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol) , the distributor also stores a `token` variable, configured via the constructor and `setToken()`, representing the specific TVS ERC‑20 (e.g. A26Z) whose unclaimed amounts should be used to calculate dividends:

- `token` is intended to be “the token inside the TVS” (e.g. A26Z).
- `stablecoin` is the stablecoin that will be paid out as dividends.
- `vesting` is the `IAlignerzVesting` implementation that holds the allocations, and `nft` is the NFT contract representing TVSs.

Dividend calculations are built on top of two functions:

- [`getUnclaimedAmounts(uint256 nftId)`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161) is supposed to compute the unclaimed TVS amount for a given TVS NFT.
- [`getTotalUnclaimedAmounts()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L127-L136) iterates over all NFTs, sums their unclaimed amounts via `getUnclaimedAmounts()` and uses this as the denominator to apportion the stablecoin pool.

The core logic in [`getUnclaimedAmounts()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161) is:

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

The intent for this function is to:

1. Retrieve the `Allocation` for `nftId` from `vesting.allocationOf(nftId)`.
2. Confirm the NFT corresponds to the token that this distributor is responsible for (A26Z).
3. Sum up any unclaimed allocation for that token.

However, the conditional check is inverted:

- **Current behavior:**  
  `if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;`

  Whenever the TVS allocation token for `nftId` **matches** the distributor’s configured `token` (e.g. both A26Z), the function immediately returns `0` and does not compute any unclaimed amount.

- **Effect:**  
  - All TVS NFTs whose `allocationOf(nftId).token` equals the distributor’s `token` (e.g. the core A26Z TVSs that this distributor is intended to serve) contribute **0** to `totalUnclaimedAmounts`.
  - Only NFTs for which `allocationOf(nftId).token` is **different** from `token` are counted.

Given that `AlignerzVesting` explicitly sets `allocationOf[nftId].token` to the project’s TVS ERC‑20, configuring `A26ZDividendDistributor` with that same TVS ERC‑20 means all relevant TVSs are systematically excluded from dividend calculations.

The highest-impact scenario is:

1. A project uses `AlignerzVesting` to create TVSs in A26Z for multiple users.
2. The protocol deploys `A26ZDividendDistributor` with:
   - `token` = A26Z TVS token,
   - `vesting` = the deployed `AlignerzVesting`,
   - `nft` = the TVS NFT contract,
   - `stablecoin` funded with the stablecoin dividend pool.
3. The owner calls `setUpTheDividends()` (or `setAmounts()` then `setDividends()`), expecting dividends to be apportioned based on unclaimed A26Z across all TVSs.
4. For each TVS NFT, `getUnclaimedAmounts(nftId)` returns `0` whenever the allocation token equals A26Z, meaning the entire set of A26Z TVSs is treated as having zero unclaimed value.
5. `totalUnclaimedAmounts` may remain zero or underrepresent the TVS base, and `dividendsOf[owner].amount` is either zero or reflective only of TVSs in some other token, not A26Z.

Thus, even with valid, non-zero A26Z vesting allocations, the corresponding TVS holders never receive dividends from `A26ZDividendDistributor`, fundamentally contradicting the intended design of rewarding these specific holders.

## Impact
HIgh. Affected TVS holders (e.g. A26Z TVS holders) never receive the dividends the protocol intends for them, causing severe mis-accounting and broken economic behavior.

## Likelihood 
Medium. The natural and documented way to configure `A26ZDividendDistributor` is to set `token` equal to the TVS token used by `AlignerzVesting` (e.g. A26Z), which makes this misconfiguration very probable in real deployments, though it requires the distributor to be used at all.

## Proof of Concept
Add the PoC to `protocol/test/AlignerzVestingProtocolTest.t.sol`, using minimal mocks to isolate the dividend distributor logic and avoid proxy/getter ABI issues.

Import these first.

```solidity
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
import {IAlignerzNFT} from "../src/interfaces/IAlignerzNFT.sol";
```

1. **Mocks**

`MockAlignerzVesting` implements `IAlignerzVesting` and always returns an `Allocation` for token A26Z with a non-zero amount:

```solidity
   contract MockAlignerzVesting is IAlignerzVesting {
    Allocation internal _alloc;

    constructor(address token_) {
        _alloc.token = Aligners26(token_);
        _alloc.amounts = new uint256[](1);
        _alloc.vestingPeriods = new uint256[](1);
        _alloc.vestingStartTimes = new uint256[](1);
        _alloc.claimedSeconds = new uint256[](1);
        _alloc.claimedFlows = new bool[](1);

        _alloc.amounts[0] = 100 ether;
        _alloc.vestingPeriods[0] = 10 days;
        _alloc.vestingStartTimes[0] = block.timestamp;
        _alloc.claimedSeconds[0] = 1 days;
        _alloc.claimedFlows[0] = false;
        _alloc.isClaimed = false;
        _alloc.assignedPoolId = 0;
    }

    function allocationOf(uint256) external view override returns (Allocation memory) {
        return _alloc;
    }
}
```

`MockAlignerzNFT` is a minimal NFT implementation matching `IAlignerzNFT`, only used to satisfy the distributor constructor:

```solidity
   contract MockAlignerzNFT is IAlignerzNFT {
    address internal _owner;

    constructor(address initialOwner) {
        _owner = initialOwner;
    }

    function mint(address to) external returns (uint256) {
        _owner = to;
        return 1;
    }

    function extOwnerOf(uint256) external view returns (address) {
        return _owner;
    }

    function burn(uint256) external {}

    function getTotalMinted() external pure returns (uint256) {
        return 1;
    }
}

```

2. **PoC test**

Add the `test_DividendDistributor_ExcludesA26ZTVS()` :

```solidity
       function test_DividendDistributor_ExcludesA26ZTVS() public {
            // This PoC uses mocks to isolate the A26ZDividendDistributor logic
            address tvsHolder = makeAddr("tvsHolder");

            // 1. Set up a mock vesting where the allocation token is A26Z and there is a non-zero unclaimed amount
            MockAlignerzVesting mockVesting = new MockAlignerzVesting(address(token));

            // 2. Set up a dummy NFT contract (only used to satisfy the constructor)
            MockAlignerzNFT mockNft = new MockAlignerzNFT(tvsHolder);

            // 3. Configure the dividend distributor for the same A26Z token
            A26ZDividendDistributor distributor = new A26ZDividendDistributor(
                address(mockVesting),
                address(mockNft),
                address(usdt),
                block.timestamp,
                90 days,
                address(token)
            );

            // 4. Because allocationOf(nftId).token == token (A26Z), the current code returns 0
            //    instead of the non-zero unclaimed amount for this TVS, excluding A26Z TVSs
            uint256 unclaimed = distributor.getUnclaimedAmounts(1);
            assertEq(unclaimed, 0, "Bug: A26Z TVS is incorrectly excluded from dividends");
        }
```

3. **How to run**

From the `protocol` directory:

```solidity
   forge test --mt test_DividendDistributor_ExcludesA26ZTVS 
```

Expected result with the current buggy implementation:

   - The test **passes**, confirming that:
     - `MockAlignerzVesting.allocationOf(1)` provides a non-zero TVS amount in A26Z.
     - `A26ZDividendDistributor.getUnclaimedAmounts(1)` returns `0` when `token` is set to A26Z.
   - This shows that TVSs whose token equals the configured `token` (A26Z) are excluded from dividend calculations.

   After applying the recommended fix (see below), the test should begin to **fail**, as `unclaimed` will then be non-zero, matching the allocation.

## Recommendation
The simplest and most direct fix is to invert the conditional in `getUnclaimedAmounts()` so that TVS NFTs are included when their allocation token **matches** the distributor’s configured `token`, and excluded otherwise.


  