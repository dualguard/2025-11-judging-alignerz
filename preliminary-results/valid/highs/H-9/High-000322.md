# [000322] Dividend distribution module unusable due to `allocationOf()` ABI mismatch
  
  ## Finding Description
The `A26ZDividendDistributor` contract expects the vesting contract to implement an interface `IAlignerzVesting` that exposes an external function `allocationOf()` returning a full `Allocation` struct, including several dynamic arrays. However, the vesting implementation `AlignerzVesting` only exposes a public state variable mapping `allocationOf` of type `mapping(uint256 => Allocation)` and relies on Solidity’s auto-generated getter.

In [`IAlignerzVesting`](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/interfaces/IAlignerzVesting.sol):

- `Allocation` is defined as:
  - `uint256[] amounts`
  - `uint256[] vestingPeriods`
  - `uint256[] vestingStartTimes`
  - `uint256[] claimedSeconds`
  - `bool[] claimedFlows`
  - `bool isClaimed`
  - `IERC20 token`
  - `uint256 assignedPoolId`
- The interface declares:
  - `function allocationOf(uint256 nftId) external view returns (Allocation memory);`

In [`AlignerzVesting`](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol):

- The same `Allocation` struct is defined.
- The only `allocationOf` entry is:
  - `mapping(uint256 => Allocation) public allocationOf;`

Solidity’s rules for public getters on mappings of structs state that when the struct contains dynamic arrays, the auto-generated getter **omits** those array fields and only returns the non-array fields. Concretely, for `mapping(uint256 => Allocation) public allocationOf`, the generated getter has an ABI equivalent to:

```solidity
function allocationOf(uint256 nftId)
    external
    view
    returns (bool isClaimed, IERC20 token, uint256 assignedPoolId);
```

This differs from the [`IAlignerzVesting.allocationOf()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/interfaces/IAlignerzVesting.sol#L67) function, which expects to return the full `Allocation` struct including all the dynamic arrays.

Despite this discrepancy, the function selector for `allocationOf(uint256)` is computed only from the function name and parameter types, not the return types. This means:

- Calls from `A26ZDividendDistributor` to `vesting.allocationOf(nftId)` are routed to the auto-generated getter in `AlignerzVesting`.
- The callee returns ABI-encoded `(bool, address, uint256)`.
- The caller attempts to ABI-decode the return data as a full `Allocation` struct with multiple dynamic arrays, as defined in the interface.

In `A26ZDividendDistributor`, the function [`getUnclaimedAmounts()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161) uses `vesting.allocationOf(nftId)` several times:

- It checks `vesting.allocationOf(nftId).token`.
- It reads `vesting.allocationOf(nftId).amounts`, `claimedSeconds`, `vestingPeriods`, and `claimedFlows`.
- It relies on `vesting.allocationOf(nftId).amounts.length` for loop bounds.

Because the actual return value only contains three fields, the ABI decoder in `A26ZDividendDistributor` reverts when attempting to decode the truncated data into the full `Allocation` struct expected from the interface.

The highest-impact scenario is when the protocol deploys `A26ZDividendDistributor` pointing to a live `AlignerzVesting` instance and the owner calls [`setUpTheDividends()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L111-L114) or [`setAmounts()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L117-L119):

- `setUpTheDividends()` internally calls `_setAmounts()` and `_setDividends()`.
- `_setAmounts()` sets `totalUnclaimedAmounts = getTotalUnclaimedAmounts()`.
- `getTotalUnclaimedAmounts()` iterates over all NFTs and calls `getUnclaimedAmounts(i)` for each.
- `getUnclaimedAmounts()` tries to read `vesting.allocationOf(nftId)` as a full `Allocation`.

At the first `vesting.allocationOf(nftId)` decode, the ABI mismatch causes a revert. As a result:

- `getUnclaimedAmounts()` reverts.
- `getTotalUnclaimedAmounts()` reverts.
- `_setAmounts()` and therefore `setUpTheDividends()` and `setAmounts()` cannot succeed.
- Dividend setup is impossible, rendering the dividend distribution module non-functional.

This is a hard functional bug in the integration between `AlignerzVesting` and `A26ZDividendDistributor`, not a design choice, since the distributor’s logic explicitly depends on the arrays within `Allocation` and the interface is written under the assumption that they are remotely readable.


## Impact
High, The ABI mismatch deterministically bricks the dividend distribution flow: `setUpTheDividends()` and related functionality always revert, resulting in a severe and permanent loss of availability for the entire dividend module.

## Likelihood 
High, Whenever `A26ZDividendDistributor` is deployed with a real `AlignerzVesting` instance and the protocol attempts to configure dividends using the provided functions, the failure is guaranteed due to the ABI mismatch; there are no special conditions required.

## Proof of Concept
Add the PoC to `protocol/test/AlignerzVestingProtocolTest.t.sol` as the function `test_DividendDistributorAbiMismatch()`.

Import this:
```solidity
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
```
Add this PoC:

```solidity
    function test_DividendDistributorAbiMismatch() public {
        A26ZDividendDistributor distributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(usdt),
            block.timestamp,
            30 days,
            address(token)
        );

        vm.expectRevert();
        distributor.getUnclaimedAmounts(0);
    }
```

High-level behavior of the PoC:

- Uses the existing `setUp()` in `AlignerzVestingProtocolTest` to deploy:
  - `AlignerzVesting` via UUPS proxy.
  - `Aligners26` token.
  - `AlignerzNFT`.
  - `MockUSD` as the stablecoin.
- Deploys `A26ZDividendDistributor` with:
  - `_vesting` = `address(vesting)`
  - `_nft` = `address(nft)`
  - `_stablecoin` = `address(usdt)`
  - `_startTime` = `block.timestamp`
  - `_vestingPeriod` = `30 days`
  - `_token` = `address(token)`
- Then sets:
  - `vm.expectRevert();`
- And calls:
  - `distributor.getUnclaimedAmounts(0);`

Because of the ABI mismatch on `allocationOf()`, the call to `getUnclaimedAmounts(0)` reverts when the return data from `AlignerzVesting`’s getter is decoded as the full `Allocation` struct. The test passes only if the revert occurs, conclusively demonstrating the bug.

To run just the PoC:

```bash
cd protocol
forge clean
forge build
forge test --mt test_DividendDistributorAbiMismatch
```

## Recommendation
Align the vesting implementation with the `IAlignerzVesting` interface by providing an explicit `allocationOf()` function that returns the full `Allocation` struct, and avoid relying on the auto-generated getter for the `mapping(uint256 => Allocation)`.


  