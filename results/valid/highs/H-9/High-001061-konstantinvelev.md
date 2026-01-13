# [001061] [H] ABI mismatch for allocationOf causes hard revert in Dividend Distributor
  
  ### Summary
`A26ZDividendDistributor` calls `vesting.allocationOf(nftId)` through the `IAlignerzVesting` interface expecting to read the entire `Allocation` struct (including dynamic arrays). However, in `AlignerzVesting` the public mapping `mapping(uint256 => Allocation) public allocationOf;` exposes Solidity’s auto-getter which only returns the scalar fields of the struct (e.g., `isClaimed`, `token`, `assignedPoolId`). The ABI expectation in the interface does not match the implementation’s auto-getter return, so decoding reverts at runtime. This breaks `getUnclaimedAmounts()` and any path that touches it. This is big issue as breaks contracts intended behavior and basically can block the proper using of it's devidents.

### Root Cause
- Interface declares a function returning the full `Allocation` (with dynamic arrays).
- Implementation uses a public mapping auto-getter that returns only fixed-size scalars, not arrays.
- Calling through the interface decodes the returned tuple as a full struct → decoding mismatch → revert.

Relevant code:

```
protocol/src/interfaces/IAlignerzVesting.sol
function allocationOf(uint256 nftId) external view returns (Allocation memory);

protocol/src/contracts/vesting/AlignerzVesting.sol
mapping(uint256 => Allocation) public allocationOf; // auto-getter returns only scalars

protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol
// getUnclaimedAmounts() reads vesting.allocationOf(nftId).token → triggers ABI mismatch revert
```

### Internal Pre-conditions
- At least one NFT `nftId` exists with an `Allocation` stored in `allocationOf[nftId]`.

### External Pre-conditions
- `A26ZDividendDistributor.getUnclaimedAmounts(nftId)` is called on-chain.

### Attack Path / Failure Path
1. Mint a TVS NFT (reward or bidding path).
2. Call `A26ZDividendDistributor.getUnclaimedAmounts(nftId)`.
3. Function internally calls `vesting.allocationOf(nftId)` through the interface and tries to read `.token`.
4. Return data doesn’t match the interface ABI (only scalars returned by auto-getter) → revert.

### Impact
- High-severity availability break for dividend logic: `getUnclaimedAmounts` hard-reverts.
- Any admin flows that use it (directly or indirectly) cannot proceed.

### PoC
```solidity
    function test_ABI_Mismatch_AllocationGetter_Revert_Simple() public {
        // Setup a Reward project and mint five NFTs via reward path (no merkle complexity)
        address[5] memory kolsFixed = [bidders[4], bidders[5], bidders[6], bidders[7], bidders[8]];
        uint256 tvsAmount = 100 ether;
        uint256 vestingPeriod = 90 days;

        // Launch reward project
        vm.startPrank(projectCreator);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 7 days);

        // Allocate TVS to five KOLs
        address[] memory kols = new address[](5);
        uint256[] memory amounts = new uint256[](5);
        for (uint256 i; i < 5; i++) {
            kols[i] = kolsFixed[i];
            amounts[i] = tvsAmount;
        }
        vesting.setTVSAllocation(0, tvsAmount * 5, vestingPeriod, kols, amounts);
        vm.stopPrank();

        // All KOLs claim TVS -> NFTs minted (ids 1..5)
        for (uint256 i; i < 5; i++) {
            vm.prank(kolsFixed[i]);
            vesting.claimRewardTVS(0);
        }

        // nft ids in ERC721A start from 1; basic sanity for first/last
        assertEq(nft.ownerOf(1), kolsFixed[0]);
        assertEq(nft.ownerOf(5), kolsFixed[4]);

        // Deploy distributor
        A26ZDividendDistributor dist = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(usdt),
            block.timestamp,
            vestingPeriod,
            address(token)
        );

        // Expect revert due to ABI mismatch on vesting.allocationOf(nftId) decoding
        vm.expectRevert();
        dist.getUnclaimedAmounts(1);

        // With five NFTs minted, distributor loops will reach tokenId=1 -> ABI mismatch -> revert
        vm.expectRevert();
        dist.setAmounts();

        vm.expectRevert();
        dist.setUpTheDividends();
    }
```

```
forge test --mt test_ABI_Mismatch_AllocationGetter_Revert_Simple -vv
```

### Mitigation
- Provide an explicit view function in `AlignerzVesting` that returns the full `Allocation` (including arrays), e.g. `getAllocation(nftId)`; update `IAlignerzVesting` and have the distributor use it.

Concrete steps (Option 1 — preferred)
1) File: `protocol/src/interfaces/IAlignerzVesting.sol`
   - Add a new accessor that returns the entire allocation (arrays + scalars):
```
function getAllocation(uint256 nftId)
    external
    view
    returns (
        uint256[] memory amounts,
        uint256[] memory vestingPeriods,
        uint256[] memory vestingStartTimes,
        uint256[] memory claimedSeconds,
        bool[] memory claimedFlows,
        bool isClaimed,
        IERC20 token,
        uint256 assignedPoolId
    );
```

2) File: `protocol/src/contracts/vesting/AlignerzVesting.sol`
   - Implement the view to read from storage and return all fields:
```
function getAllocation(uint256 nftId)
    external
    view
    returns (
        uint256[] memory amounts,
        uint256[] memory vestingPeriods,
        uint256[] memory vestingStartTimes,
        uint256[] memory claimedSeconds,
        bool[] memory claimedFlows,
        bool isClaimed,
        IERC20 token,
        uint256 assignedPoolId
    )
{
    Allocation memory a = allocationOf[nftId];
    return (
        a.amounts,
        a.vestingPeriods,
        a.vestingStartTimes,
        a.claimedSeconds,
        a.claimedFlows,
        a.isClaimed,
        a.token,
        a.assignedPoolId
    );
}
```
   - Note: this is a read-only function; it doesn’t change storage layout and is safe for UUPS upgrade.

3) File: `protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol`
   - Replace any usage of `vesting.allocationOf(nftId)` with a call to `vesting.getAllocation(nftId)` and destructure results, for example:
```
(uint256[] memory amounts,
 uint256[] memory vestingPeriods,
 uint256[] memory vestingStartTimes,
 uint256[] memory claimedSeconds,
 bool[] memory claimedFlows,
 /*isClaimed*/,
 /*token*/,
 /*assignedPoolId*/
) = vesting.getAllocation(nftId);
```
   - Use the destructured arrays/values in subsequent calculations.
  