# [000422] Uninitialized unclaimedAmountsIn Mapping Causes Complete Dividend Distribution Failure
  
  The A26ZDividendDistributor contract’s `_setDividends()` logic calculates per-NFT dividend allocations using `unclaimedAmountsIn[nftId]`, but `unclaimedAmountsIn` is never written anywhere in the contract. This results in all dividend calculations evaluating to zero (0 * stablecoinAmountToDistribute / totalUnclaimedAmounts = 0), preventing any user from receiving dividends. This effectively prevents any dividend distribution — funds appear locked and users receive nothing when they call `claimDividends()`.


## Proof of concept;

* Steps: 
  * Mint NFT (ID 0) to Alice.
  * Mint 1000 USDC to Alice and ensure funds exist in the system.
  * Warp to distribution start time and call `setUpTheDividends()` as owner.
  * Alice calls `claimDividends()` and receives `0` stablecoin.
  * Reading `distributor.dividendsOfUser(nftowner)` returns `Dividend.amount == 0`.
 
* Foundry Tests;
```
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "forge-std/console2.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";
import {CompleteMerkle} from "murky/src/CompleteMerkle.sol";

contract Mine is Test {
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public stablecoin;
    AlignerzVesting public vesting;
    A26ZDividendDistributor public distributor;

    uint256 startTime = block.timestamp + 100; 
    uint256 vestingPeriod = 90 days;
    address public owner;
    address public alice;

    function setUp() public {
        owner = address(this);
        alice = makeAddr("alice");
        vm.deal(alice, 100 ether);

        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");
        vesting = new AlignerzVesting();
        vesting.initialize(address(nft));

        stablecoin = new MockUSD();
        distributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(stablecoin),
            startTime,
            vestingPeriod,
            address(token)
        );
    }

    function testPOC() public {
        // Mint NFT to Alice (NFT ID = 0)
        nft.mint(alice);
        // Fund distributor with stablecoin
        stablecoin.mint(address(alice), 1000e6); // 1000 USDC

        console2.log("1: Mint NFT to Alice -NFT ID = 0");
        console2.log("2: Deposit 1000 USDC to user (simulating protocol funding)\n");

        vm.warp(startTime + 1 days);

        // Run dividends setup
        vm.prank(owner);
        distributor.setUpTheDividends();

        vm.startPrank(alice);
        uint256 aliceStablecoinBalance = stablecoin.balanceOf(alice);
        uint256 nftCount = nft.getTotalMinted();

        console2.log("3: Total NFTs minted:", nftCount);
        console2.log("4: Alice stablecoin balance AFTER claiming (should increase if dividends worked):");
        console2.log(aliceStablecoinBalance);

        (address nftowner, bool isOwned) = distributor.safeOwnerOf(nftCount);
        console2.log("5: NFT Owner address:", nftowner);
        console2.log("6: NFT Exists in distributor view:", isOwned);

        // Read dividend struct
        A26ZDividendDistributor.Dividend memory div = distributor.dividendsOfUser(nftowner);

        console2.log("\nDividend struct for Alice:");
        console2.log("7: Dividend amount (BUG: expected >0):", div.amount);
        console2.log("8: Claimed seconds:", div.claimedSeconds);
        distributor.claimDividends();


        console2.log("9: RESULT: User receives ZERO dividends despite contract being funded ===");
        console2.log("ROOT CAUSE: unclaimedAmountsIn[] is never populated -all payouts = 0 ===\n");
    }
}
```
* Conclusion: The test shows the distribution algorithm produces zero allocations because `unclaimedAmountsIn` was never populated.

## Root cause

* **Missing initialization/assignment**: The mapping `unclaimedAmountsIn` is declared and used, but never assigned any nonzero values anywhere in the contract. There is no logic to:

  * calculate each NFT’s entitlement (e.g., based on weight, vesting, share, or token balance), or
  * populate `unclaimedAmountsIn` during setup/mint/vesting events.
* **Dependency on an external state that never exists**: `_setDividends()` assumes `unclaimedAmountsIn` and `totalUnclaimedAmounts` have been previously computed. Without that precomputation, the formula collapses to zero.
* **Lack of checks / assertions**: No guard clauses verify `totalUnclaimedAmounts > 0` or that `unclaimedAmountsIn` contains valid data prior to division; this allows the faulty path to proceed silently.

# Impact

* **Funds at risk of being permanently locked**: Stablecoins deposited for distribution will not reach recipients via the normal distribution path.
* **Complete failure of dividends feature**: All users receive zero dividends regardless of contract stablecoin balance.
* **Operational risk**: Users and integrators will observe funds not being distributed; this damages usability and trust, and requires a contract patch or migration.
* **Potential financial loss**: If funds cannot be recovered or reallocated because on-chain bookkeeping is wrong, users may lose access to rightful payouts.


## Recommended fixes (detailed)

Below are corrective approaches ordered from minimal patch to robust redesign. Choose the one consistent with the contract's intent (how NFT entitlements should be computed).

### 1) Minimal fix — populate `unclaimedAmountsIn` before `_setDividends()`

* Add logic to compute and populate `unclaimedAmountsIn[nftId]` for every relevant NFT prior to calling `_setDividends()`.
* Compute `totalUnclaimedAmounts` as the sum of all `unclaimedAmountsIn[nftId]` values.
* Add defensive checks:

  ```solidity
  require(totalUnclaimedAmounts > 0, "No unclaimed amounts available");
  ```
* Example (pseudocode):

  ```solidity
  function setUpTheDividends() external onlyOwner {
      // compute per-NFT unclaimed amounts
      totalUnclaimedAmounts = 0;
      for (uint i = 0; i < totalMinted; i++) {
          uint256 amt = computeUnclaimedForNFT(i); // implement according to spec
          unclaimedAmountsIn[i] = amt;
          totalUnclaimedAmounts += amt;
      }
      require(totalUnclaimedAmounts > 0, "total unclaimed is zero");
      _setDividends();
  }
  ```
* `computeUnclaimedForNFT` must be implemented according to business rules (weight per NFT, vesting schedule, token holdings, or fixed share).

### 2) Preferred robust approach — compute on the fly and avoid stale mapping

* Instead of relying on a stored mapping, compute each NFT’s allocation dynamically at distribution time (this avoids synchronization bugs).
* Example:

  ```solidity
  for (uint i = 0; i < totalMinted; i++) {
      uint256 nftWeight = computeWeight(i);
      // accumulate weights first to get totalWeight
  }
  for (uint i = 0; i < totalMinted; i++) {
      dividendsOf[owner].amount += (computeWeight(i) * stablecoinAmountToDistribute) / totalWeight;
  }
  ```
* Benefits: no need to maintain `unclaimedAmountsIn` across transfers/claims; computation is consistent with current state.

### 3) Add invariants and telemetry

* Emit events during distribution to record per-NFT allocations:

  ```solidity
  event DividendAllocated(uint256 indexed nftId, address indexed owner, uint256 amount);
  ```
* After allocation, emit `DividendDistributed(totalAllocated)` and store distributed totals for auditability.

### 4) Guard against division by zero and rounding issues

* Prevent `totalUnclaimedAmounts == 0` from proceeding.
* Use safe arithmetic (Solidity ^0.8.0 reverts on overflow, but be explicit if you depend on libraries).
* Consider distributing any remainder (from integer division) to the contract owner or pro rata across NFTs to avoid locked dust.
  