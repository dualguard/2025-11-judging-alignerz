# [000009] Missing loop increments in vesting and fees logic will block dividend setup and TVS split/merge operations
  
  ### Summary
Missing loop increments in [`A26ZDividendDistributor::getUnclaimedAmounts`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161) and [`FeesManager::calculateFeeAndNewAmountForOneTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174) will cause a denial of service of dividend setup and TVS management for TVS holders and the protocol as any caller trying to set up dividends or split/merge TVSs over non-empty vesting flows will always hit out-of-gas infinite loops and revert.

### Root Cause

In `A26ZDividendDistributor::getUnclaimedAmounts` the contract iterates vesting flows with a manual-increment `for` loop, but uses `continue` statements that skip the increment, causing an infinite loop whenever an early branch is hit:

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
    uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
    uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
    bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
    uint256 len = vesting.allocationOf(nftId).amounts.length;

@>  for (uint256 i; i < len; ) {
@>      if (claimedFlows[i]) continue;
@>      if (claimedSeconds[i] == 0) {
@>          amount += amounts[i];
@>          continue;
        }
        uint256 claimedAmount = (claimedSeconds[i] * amounts[i]) / vestingPeriods[i];
        uint256 unclaimedAmount = amounts[i] - claimedAmount;
        amount += unclaimedAmount;
        unchecked {
@>          ++i;
        }
    }

    unclaimedAmountsIn[nftId] = amount;
}
```

When either `claimedFlows[i]` is `true` or `claimedSeconds[i] == 0`, the loop executes a `continue` before reaching the `++i`, so the same index is re-evaluated forever until gas runs out and the transaction reverts. Because `claimedSeconds[i]` is initially `0` for all new flows, this bug is triggered on the very first iteration for any allocation with at least one flow, effectively making `getUnclaimedAmounts` unusable for real vesting data.

Additionally, in `FeesManager::calculateFeeAndNewAmountForOneTVS` the contract iterates over vesting flows using a manual-increment `for` loop but never advances the loop index, so `i` is always re-evaluated and the loop never terminates:

```solidity
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    // ...
@>  for (uint256 i; i < length; ) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
        // missing increment of i; the loop never progresses
    }
}
```

Because there is no `i++` or equivalent increment inside this loop body, the exit condition `i < length` is never invalidated when `length > 0`. Any call that reaches `calculateFeeAndNewAmountForOneTVS` with a non-empty `amounts` array will therefore consume all gas and revert, permanently preventing use of the helper from its callers.

### Internal Pre-conditions

Dividend setup path (`A26ZDividendDistributor::getUnclaimedAmounts`):
1. At least one NFT ID processed by `A26ZDividendDistributor::getUnclaimedAmounts` has a vesting allocation with `amounts.length > 0`.
2. For that allocation, either `claimedFlows[i] == true` or `claimedSeconds[i] == 0` (the default initial value) for some `i < len`.
3. `A26ZDividendDistributor::getUnclaimedAmounts` is invoked directly or indirectly via `A26ZDividendDistributor::getTotalUnclaimedAmounts` during `setUpTheDividends`, `setAmounts` or `setDividends`.

TVS split/merge path (`FeesManager::calculateFeeAndNewAmountForOneTVS`):
1. A TVS allocation with `amounts.length > 0` exists for the TVS being split or merged.
2. `AlignerzVesting::mergeTVS` or `AlignerzVesting::splitTVS` is called on that TVS, forwarding its `amounts` and `nbOfFlows` to `FeesManager::calculateFeeAndNewAmountForOneTVS`.

### External Pre-conditions

None

### Attack Path

Dividend setup path (`A26ZDividendDistributor::getUnclaimedAmounts`):
1. The protocol configures `A26ZDividendDistributor` with a vesting project where TVS NFTs have non-empty vesting allocations and sends stablecoins to the distributor.
2. The owner calls `setUpTheDividends` (or `setAmounts` followed by `setDividends`) to initialize dividends for all TVS holders.
3. Inside `_setAmounts`, `getTotalUnclaimedAmounts` iterates over NFT IDs and, for each owned NFT, calls `getUnclaimedAmounts(nftId)`.
4. In `getUnclaimedAmounts`, the loop enters with `i = 0`, hits a branch where either `claimedFlows[0] == true` or `claimedSeconds[0] == 0`, executes a `continue` without incrementing `i`, and then re-evaluates index `0` indefinitely until gas is exhausted and the transaction reverts, causing dividend setup to fail.

TVS split/merge path (`FeesManager::calculateFeeAndNewAmountForOneTVS`):
1. A user holds a TVS with at least one vesting flow (`amounts.length > 0`) and calls `AlignerzVesting::mergeTVS` to merge it, or `AlignerzVesting::splitTVS` to split it.
2. Inside these functions, the protocol computes per-flow fees by calling `FeesManager::calculateFeeAndNewAmountForOneTVS` with the TVS `amounts` and `nbOfFlows`.
3. The loop in `calculateFeeAndNewAmountForOneTVS` enters with `i = 0` and never increments `i`, so it repeatedly processes index `0` until gas is exhausted and the transaction reverts, preventing the merge or split from completing.

### Impact

Dividend setup path (`A26ZDividendDistributor::getUnclaimedAmounts`):
The TVS NFT holders suffer an effective loss of access to dividends, as any dividend setup or refresh that relies on `getUnclaimedAmounts` reverts and up to the full `stablecoinAmountToDistribute` (the distributor’s stablecoin balance) cannot be distributed according to the intended vesting logic, remaining recoverable only via owner-admin withdrawals.

TVS split/merge path (`FeesManager::calculateFeeAndNewAmountForOneTVS`):
The TVS holders cannot execute `mergeTVS` or `splitTVS` on positions with non-empty vesting flows, so they are unable to restructure or rebalance their vested allocations and any protocol features that rely on TVS split/merge are effectively bricked.

### PoC

_No response_

### Mitigation

Ensure that the loop index is always incremented in both places. For the dividend path, increment `i` on all branches before any `continue`:

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    // ...
    uint256 len = vesting.allocationOf(nftId).amounts.length;

    for (uint256 i; i < len; ) {
        if (claimedFlows[i]) {
@>          unchecked { ++i; }
            continue;
        }

        if (claimedSeconds[i] == 0) {
            amount += amounts[i];
@>          unchecked { ++i; }
            continue;
        }

        uint256 claimedAmount = (claimedSeconds[i] * amounts[i]) / vestingPeriods[i];
        uint256 unclaimedAmount = amounts[i] - claimedAmount;
        amount += unclaimedAmount;
        unchecked { ++i; }
    }
}
```

For the TVS fee helper, rely on a canonical `for` loop that increments `i` on every iteration:

```solidity
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    // ...
    for (uint256 i; i < length; ++i) {
        uint256 flowFee = calculateFeeAmount(feeRate, amounts[i]);
        feeAmount += flowFee;
        newAmounts[i] = amounts[i] - flowFee;
    }
}
```
  