# [000324] Dividend distribution DoS via infinite loop in `getUnclaimedAmounts()`
  
  ## Finding Description
The `A26ZDividendDistributor` contract is responsible for computing the USD value of unclaimed TVS tokens and distributing stablecoin dividends accordingly. This logic hinges on [`getTotalUnclaimedAmounts()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L127-L136) and [`getUnclaimedAmounts()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161).

In [`A26ZDividendDistributor.sol::getTotalUnclaimedAmounts()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L127-L136) :

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

[`getUnclaimedAmounts()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161) is implemented as:

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;

    uint256[] memory amounts       = vesting.allocationOf(nftId).amounts;
    uint256[] memory claimedSeconds= vesting.allocationOf(nftId).claimedSeconds;
    uint256[] memory vestingPeriods= vesting.allocationOf(nftId).vestingPeriods;
    bool[] memory claimedFlows     = vesting.allocationOf(nftId).claimedFlows;
    uint256 len = vesting.allocationOf(nftId).amounts.length;

    for (uint i; i < len;) {
        if (claimedFlows[i]) continue;
        if (claimedSeconds[i] == 0) {
            amount += amounts[i];
            continue;
        }
        uint256 claimedAmount   = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
        uint256 unclaimedAmount = amounts[i] - claimedAmount;
        amount += unclaimedAmount;
        unchecked { ++i; }
    }
    unclaimedAmountsIn[nftId] = amount;
}
```

The intent is:

- For each NFT (`nftId`), compute the USD value of all unclaimed token “flows”.
- Skip fully-claimed flows where `claimedFlows[i] == true`.
- If `claimedSeconds[i] == 0`, the flow is entirely unclaimed, so add `amounts[i]`.
- Otherwise, compute the remaining unclaimed portion as `amounts[i] - claimedAmount`.

Root cause: loop counter never incremented on `continue` branches

The `for` loop uses a manual increment pattern:
- Loop header: `for (uint i; i < len;) { ... }`
- Counter increment only appears at the bottom in `unchecked { ++i; }`.

However, there are two early `continue` statements:

- `if (claimedFlows[i]) continue;`
- `if (claimedSeconds[i] == 0) { amount += amounts[i]; continue; }`

In both branches, the code jumps to the next iteration of the loop **without executing** the `unchecked { ++i; }` block.

As a result, if `claimedFlows[i] == true`, the loop re-enters with the **same** `i` on every iteration. If `claimedSeconds[i] == 0`, it adds `amounts[i]` once, then keeps re-evaluating the same `i` forever.

In both cases, `i` is never incremented and the loop never reaches `i == len`. This leads to an **infinite loop** (practically, an out-of-gas revert) as soon as one of these conditions holds for any flow.

The vesting contract `AlignerzVesting` creates and manages `Allocation` records for NFTs. On allocation creation (both reward projects and bidding projects), the initial state for each flow is:

- `claimedSeconds[i] == 0`
- `claimedFlows[i] == false`

For example, in `AlignerzVesting.sol` when an NFT is minted for an accepted bid:

[AlignerzVesting.sol#L879-L887](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L879-L887)
```solidity
biddingProject.allocations[nftId].amounts.push(amount);
biddingProject.allocations[nftId].vestingPeriods.push(bid.vestingPeriod);
biddingProject.allocations[nftId].vestingStartTimes.push(biddingProject.endTime);
biddingProject.allocations[nftId].claimedSeconds.push(0);
biddingProject.allocations[nftId].claimedFlows.push(false);
biddingProject.allocations[nftId].assignedPoolId = poolId;
biddingProject.allocations[nftId].token = biddingProject.token;
allocationOf[nftId] = biddingProject.allocations[nftId];
```

Thus, for a newly created TVS NFT with at least one flow:
- The very first flow has `claimedSeconds[0] == 0` and `claimedFlows[0] == false`.

When the dividend distributor is configured **not** to early-return (i.e. `address(token) != address(vesting.allocationOf(nftId).token)`), `getUnclaimedAmounts(nftId)` will:

- Enter the loop with `i = 0`.
- Evaluate `if (claimedFlows[0])` as false.
- Evaluate `if (claimedSeconds[0] == 0)` as true, add `amounts[0]` to `amount`, then hit `continue;` **without incrementing `i`**.
- Re-enter the loop with `i = 0` again, and repeat forever.

This is a normal, expected vesting state (a fresh allocation with no tokens claimed yet), so the infinite loop is reachable under realistic conditions.

The highest-impact scenario occurs like, the core owner functions that configure the dividends depend on `getTotalUnclaimedAmounts()`:

```solidity
function _setAmounts() internal {
    stablecoinAmountToDistribute = stablecoin.balanceOf(address(this));
    totalUnclaimedAmounts        = getTotalUnclaimedAmounts();
    emit amountsSet(stablecoinAmountToDistribute, totalUnclaimedAmounts);
}

function setAmounts() public onlyOwner {
    _setAmounts();
}

function setUpTheDividends() external onlyOwner {
    _setAmounts();
    _setDividends();
}
```

Call chain in the highest-impact scenario:

1. There exists at least one NFT with a non-trivial vesting allocation:
   - At least one flow where `claimedSeconds[i] == 0` and `claimedFlows[i] == false` (a fresh, unclaimed vesting flow).
2. The dividend distributor is configured such that:
   - `address(token) != address(vesting.allocationOf(nftId).token)`
   - This can happen via the constructor parameter `_token` or via `setToken()`.
3. The owner calls `setAmounts()` or `setUpTheDividends()`:
   - `_setAmounts()` calls `getTotalUnclaimedAmounts()`.
   - `getTotalUnclaimedAmounts()` iterates over minted NFT IDs and, for any live NFT, calls `getUnclaimedAmounts(nftId)`.
   - For a fresh allocation, `getUnclaimedAmounts(nftId)` enters the loop at `i = 0`, hits `claimedSeconds[0] == 0`, and loops forever (effectively an out-of-gas revert).

Impact in this scenario:

- Every call to `setAmounts()` / `setUpTheDividends()` **reverts** as soon as there is at least one unclaimed flow and the token configuration bypasses the initial short-circuit, rendering the dividend configuration logic unusable.
- This is a full **Denial of Service** of the dividend distribution mechanism.

## Attack Path
1. Protocol deploys `A26ZDividendDistributor` with:
   - `_vesting` pointing to the live `AlignerzVesting` contract.
   - `_nft` pointing to the `AlignerzNFT` contract.
   - `_stablecoin` set to the stablecoin used for dividends.
   - `_token` set to **any** ERC20 address different from `vesting.allocationOf(nftId).token` (or later changed via `setToken()`).
2. Normal protocol operation creates at least one TVS NFT with an allocation:
   - E.g. users participate in a bidding project, project owner finalizes bids, and a winner claims their NFT via `vesting.claimNFT()`.
   - The resulting allocation has at least one flow with `claimedSeconds[i] == 0` and `claimedFlows[i] == false`.
3. The owner later attempts to configure dividends:
   - Calls `setAmounts()` or `setUpTheDividends()`.
4. `setAmounts()` executes `_setAmounts()`:
   - `getTotalUnclaimedAmounts()` iterates over NFT IDs, encounters an owned NFT, and calls `getUnclaimedAmounts(nftId)`.
5. In `getUnclaimedAmounts(nftId)` for that NFT:
   - The guard `if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;` does **not** trigger because `token` was set to a different address.
   - The loop runs from `i = 0`. For the first flow:
     - `claimedFlows[0] == false` so the first `if` is skipped.
     - `claimedSeconds[0] == 0` so the second `if` adds `amounts[0]` and then `continue;` without incrementing `i`.
   - The same iteration is executed indefinitely until out of gas.
6. The transaction reverts due to out-of-gas, and `setAmounts()` / `setUpTheDividends()` becomes unusable, effectively DoSing the dividend configuration.


## Impact
Medium, This causes a complete denial of service of the dividend distribution mechanism (`setAmounts()` / `setUpTheDividends()` cannot succeed), but does not directly enable theft or misappropriation of funds.

## Likelihood
Medium, The infinite loop becomes active whenever there is at least one unclaimed vesting flow and the dividend distributor’s `token` is configured to differ from the TVS token in allocations, a configuration that is plausible given the presence of `setToken()` and multi-token support.

## Proof of Concept
Add the PoC test to `protocol/test/AlignerzVestingProtocolTest.t.sol` as `test_DividendDistributor_getUnclaimedAmounts_infinite_loop()`

First import this:
```diff
+ import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
```
```solidity
    function test_DividendDistributor_getUnclaimedAmounts_infinite_loop() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vesting.addUsersToWhitelist(bidders, PROJECT_ID);
        vm.stopPrank();

        address bidder0 = bidders[0];
        address bidder1 = bidders[1];

        vm.startPrank(bidder0);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        vm.startPrank(bidder1);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        BidInfo[] memory bids = new BidInfo[](2);
        bids[0] = BidInfo({
            bidder: bidder0,
            amount: BIDDER_USD,
            vestingPeriod: 90 days,
            poolId: 0,
            accepted: true
        });
        bids[1] = BidInfo({
            bidder: bidder1,
            amount: BIDDER_USD,
            vestingPeriod: 90 days,
            poolId: 0,
            accepted: true
        });

        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = generateMerkleProofs(bids, 0);

        vm.startPrank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);
        vm.stopPrank();

        vm.startPrank(bidder0);
        vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[bidder0]);
        vm.stopPrank();

        vm.startPrank(bidder1);
        vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[bidder1]);
        vm.stopPrank();

        A26ZDividendDistributor distributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(usdt),
            block.timestamp,
            90 days,
            address(1)
        );

        vm.deal(address(distributor), 1 ether);

        vm.expectRevert();
        distributor.setAmounts();
    }
```

High-level behavior of the test:

1. Uses the existing setup in `AlignerzVestingProtocolTest` to:
   - Deploy `Aligners26` (`token`), `MockUSD` (`usdt`), `AlignerzNFT` (`nft`), and an upgradeable `AlignerzVesting` proxy.
   - Configure `nft` to allow the vesting contract as minter.
   - Create bidders and fund them with `MockUSD`.
2. As `projectCreator`:
   - Calls `vesting.setVestingPeriodDivisor(1)`.
   - Launches a bidding project with `token` and `usdt`.
   - Creates one pool and whitelists all bidders.
3. Two bidders (`bidder0` and `bidder1`) place bids in the project.
4. Off-chain style allocation is simulated in the test:
   - A `BidInfo[]` with both bidders marked `accepted = true` for `poolId = 0`.
   - `generateMerkleProofs(bids, 0)` produces a Merkle root and stores proofs for both bidders.
5. As `projectCreator`:
   - Calls `vesting.finalizeBids()` with that root, closing the project and setting the allocation Merkle root.
6. As each bidder:
   - Calls `vesting.claimNFT()` with their proof, minting two NFTs (IDs `1` and `2`) that each have an allocation with `claimedSeconds[0] == 0` and `claimedFlows[0] == false`.
7. The test then deploys an `A26ZDividendDistributor` instance with:
   - `_vesting = address(vesting)`
   - `_nft = address(nft)`
   - `_stablecoin = address(usdt)`
   - `_startTime = block.timestamp`
   - `_vestingPeriod = 90 days`
   - `_token = address(1)` (a dummy token, intentionally different from the TVS token used in allocations to bypass the early-return).
8. The test funds the distributor with some ether (for gas, non-critical) and sets:

   - `vm.expectRevert();`
   - Then calls `distributor.setAmounts();`

9. Observed behavior:
   - `setAmounts()` → `_setAmounts()` → `getTotalUnclaimedAmounts()`:
     - `nft.getTotalMinted()` returns `2` (NFT IDs `1` and `2`).
     - For `i = 0`, `safeOwnerOf(0)` detects no token and skips.
     - For `i = 1` (and/or `i = 2`), `safeOwnerOf` returns a valid owner, so `getUnclaimedAmounts(nftId)` is called.
   - Inside `getUnclaimedAmounts(nftId)`, because the allocation is fresh:
     - `claimedSeconds[0] == 0`
     - `claimedFlows[0] == false`
   - The code hits `if (claimedSeconds[0] == 0) { ...; continue; }` without incrementing `i`, creating an infinite loop and causing the transaction to revert due to out-of-gas.
   - Because `vm.expectRevert()` is set, the test passes and demonstrates the bug.



How to run the PoC:

1. From the `protocol` directory:

   ```solidity
   forge clean
   forge build
   forge test --mt test_DividendDistributor_getUnclaimedAmounts_infinite_loop
   ```

2. Expected outcome:
   - Compilation succeeds.
   - `test_DividendDistributor_getUnclaimedAmounts_infinite_loop()` passes, indicating that `setAmounts()` reverted as expected due to the infinite loop in `getUnclaimedAmounts()`.

## Recommendation
Fix the loop in `getUnclaimedAmounts()` so that the index `i` is incremented on **every** iteration, including when `continue;` is used. A minimal patch is to increment `i` in each branch that early-continues:

- Increment `i` when skipping claimed flows.
- Increment `i` after counting fully unclaimed flows (`claimedSeconds[i] == 0`).
- Keep the existing increment for the general case.

  