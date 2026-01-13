# [000463] The `getUnclaimedAmounts` function in `A26ZDividendDistributor` incorrectly tries to fetch dynamic arrays from `allocationOf` mapping which reverts the transaction completely
  
  ### Summary

The function `getUnclaimedAmounts(uint256 nftId)` in  `A26ZDividendDistributor` contract attempts to read dynamic arrays (amounts, claimedSeconds, vestingPeriods, claimedFlows) directly from:
```solidity
vesting.allocationOf(nftId)
```
However, `allocationOf` is a public mapping → struct where the struct contains multiple dynamic arrays. Solidity’s auto-generated public getter for such mappings does not return dynamic array fields in the ABI. It only returns fixed-size fields (`isClaimed`, `token`, `assignedPoolId`).

Because of this, any external contract attempting to read:
```solidity
vesting.allocationOf(nftId).amounts
```
will revert at runtime due to ABI mismatch and failed decoding.

This makes `getUnclaimedAmounts()` completely unusable and causes it to revert entirely in deployed environments.

### Root Cause

Auto-generated getter for:
```solidity
mapping(uint256 => Allocation) public allocationOf;
```
**Returns only:**
- bool isClaimed
- address token
- uint256 assignedPoolId

**It does not return:**
- uint256[] amounts
- uint256[] claimedSeconds
- uint256[] vestingPeriods
- bool[] claimedFlows

Attempts to decode unavailable fields → ABI decode revert.

### Internal Pre-conditions

N/A

### External Pre-conditions

N/A

### Attack Path

N/A

### Impact

- The function reverts 100% of the time when reading allocations from the real vesting contract.
- No unclaimed amount can ever be computed, breaking:
   - user reward tracking
   - dividend calculations
   - vesting progress checks
   - distribution logic
- Any system depending on `getUnclaimedAmounts()` becomes non-functional.
- Downstream functions also revert, leading to a cascade of failures.
- Because reads occur across contracts, this creates a hard, unavoidable runtime failure — not just incorrect accounting.

**Severity: High (core vesting logic becomes unusable).**

### PoC

- To run the PoC, import the `A26ZDividendDistributor` contract in `AlignerzVestingProtocolTest.t.sol` test suite.
- Copy paste the given test in `AlignerzVestingProtocolTest.t.sol` test suite and run the PoC with command `forge test --mt test_IncorrectHandlingOfDynamicArraysInStructPoC -vvvv.`

```solidity
    function test_IncorrectHandlingOfDynamicArraysInStructPoC() public {
        vm.startPrank(projectCreator);
        A26ZDividendDistributor dividendDistributor = new A26ZDividendDistributor(
            address(vesting), address(nft), address(usdt), block.timestamp, 180 days, address(token)
        );

        address alice = makeAddr("Alice");
        address bob = makeAddr("bob");
        //deal tokens to the users, 1 thousand tokens to each
        deal(address(usdt), alice, 1000e6);
        deal(address(usdt), bob, 1000e6);
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // Setup the project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(
            address(token), address(usdt), block.timestamp, block.timestamp + 1000_000 days, "0x0", false
        );
        ///creating a pool with 3 thousand tokens and 1 usdt price per token
        vesting.createPool(0, 3000 ether, 1e6, false);
        vm.stopPrank();

        /// Place bids for the project
        vm.startPrank(alice);
        usdt.approve(address(vesting), 1000e6);
        vesting.placeBid(0, 1000e6, 90 days);
        vm.stopPrank();

        vm.startPrank(bob);
        usdt.approve(address(vesting), 1000e6);
        vesting.placeBid(0, 1000e6, 90 days);
        vm.stopPrank();

        //Create bind info to generate merkle proofs
        BidInfo[] memory allBids = new BidInfo[](2);

        /// Simulate off-chain allocation process
        // For simplicity, all bids are the same amount
        allBids[0] = BidInfo({bidder: alice, amount: 1000 ether, vestingPeriod: 90 days, poolId: 0, accepted: true});
        allBids[1] = BidInfo({bidder: bob, amount: 1000 ether, vestingPeriod: 90 days, poolId: 0, accepted: true});

        bytes32[] memory poolRoot = new bytes32[](1);
        poolRoot[0] = generateMerkleProofs(allBids, 0);

        //finalize bids
        vm.startPrank(projectCreator);
        vesting.finalizeBids(0, bytes32(0), poolRoot, 30 days);
        vm.stopPrank();

        ///only one user cliam nft, so total minted = 1
        vm.prank(alice);
        uint256 nftIdAlice = vesting.claimNFT(0, 0, 1000 ether, bidderProofs[alice]);
        assertEq(nft.ownerOf(nftIdAlice), alice);
        vm.prank(bob);
        uint256 nftIdBob = vesting.claimNFT(0, 0, 1000 ether, bidderProofs[bob]);
        assertEq(nft.ownerOf(nftIdBob), bob);

        //The call will revert because of how the `getUnclaimedAmounts` function handles vesting.allocationOf dynamic arrays
        vm.expectRevert();
        dividendDistributor.getTotalUnclaimedAmounts();
    }
```

**Logs:-**
```solidity
  │   ├─ [1680] AlignerzNFT::extOwnerOf(0) [staticcall]
    │   │   └─ ← [Revert] OwnerQueryForNonexistentToken()
    │   ├─ [4318] AlignerzNFT::extOwnerOf(1) [staticcall]
    │   │   └─ ← [Return] Alice: [0xBf0b5A4099F0bf6c8bC4252eBeC548Bae95602Ea]
    │   ├─ [3616] ERC1967Proxy::fallback(1) [staticcall]
    │   │   ├─ [2879] AlignerzVesting::allocationOf(1) [delegatecall]
    │   │   │   └─ ← [Return] false, Aligners26: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 0
    │   │   └─ ← [Return] false, Aligners26: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 0
    │   └─ ← [Revert] EvmError: Revert
    └─ ← [Return]
```

### Mitigation

Expose explicit getters in the vesting contract like:-
```solidity
function getAmounts(uint256 nftId) external view returns (uint256[] memory);
function getClaimedSeconds(uint256 nftId) external view returns (uint256[] memory);
function getVestingPeriods(uint256 nftId) external view returns (uint256[] memory);
function getClaimedFlows(uint256 nftId) external view returns (bool[] memory);
```
  