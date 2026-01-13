# [000258] `AlignerzVesting::_computeSplitArrays` Reverts Due to Unallocated Dynamic Arrays Inside Memory Struct
  
  ### Summary

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1088

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113-L1141

The `AlignerzVesting::_computeSplitArrays` function returns an Allocation memory struct but never allocates any of its internal dynamic arrays.
Inside the loop, the function writes to:
```solidity
alloc.amounts[j]
alloc.vestingPeriods[j]
alloc.vestingStartTimes[j]
alloc.claimedSeconds[j]
alloc.claimedFlows[j]
```
However, none of these arrays were initialized with a length. Dynamic arrays in a memory struct default to length = 0, so any index write immediately triggers an out of bounds revert. Because `AlignerzVesting::splitTVS` relies on `AlignerzVesting::_computeSplitArrays`, only `AlignerzVesting::splitTVS` is affected, and thus the `AlignerzVesting::splitTvs` function will always revert.

### Root Cause

The `AlignerzVesting::_computeSplitArrays` returns a memory Allocation struct but did not allocate storage for its internal arrays before writing to them.

Below is the affected code:
```solidity
function _computeSplitArrays(
        Allocation storage allocation,
        uint256 percentage,
        uint256 nbOfFlows
    )
        internal
        view
        returns (
            Allocation memory alloc // all arrays inside it have length 0
        )
    {
        ----------------------------------------
    for (uint256 j; j < nbOfFlows;) {
        alloc.amounts[j] = ...;             
        alloc.vestingPeriods[j] = ...;   
        alloc.vestingStartTimes[j] = ...;  
        alloc.claimedSeconds[j] = ...;  
        alloc.claimedFlows[j] = ...;
        unchecked {
                ++j;
            }
        }

        --------------------------------------------
    }
```
All of these reverts instantly.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

- `AlignerzVesting::splitTVS` calls `AlignerzVesting::_computeSplitArrays`:
```solidity
Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
```
- `AlignerzVesting::_computeSplitArrays` enters loop:
```solidity
for (uint256 j; j < nbOfFlows;) {
    alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
}
```
- `alloc.amounts` has length 0 and thus will trigger an out of bounds revert.

- `splitTVS` becomes unusable and so splitting TVS is impossible, thereby bricking a major feature of the protocol.

### Impact

- `AlignerzVesting::splitTVS` cannot be executed under any circumstances
- Users cannot split their vesting allocations
- There is complete denial of service on the function

### PoC

Paste the following test in `protocol/test/AlignerzVestingPotocolTest.t.sol` and run test with `forge test --match-test test_computeSplitArrays_uninitialized_alloc_reverts -vv`.

```solidity
function test_computeSplitArrays_uninitialized_alloc_reverts() public {
        // Use projectCreator to launch a reward project and set a TVS allocation to an EOA claimer
        vm.prank(projectCreator);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 1);

        address claimer = makeAddr("claimer");

        address[] memory kols = new address[](1);
        kols[0] = claimer;
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 100 ether;
        uint256 total = amounts[0];

        // projectCreator funds the allocation (they have tokens and approved in setUp)
        vm.prank(projectCreator);
        vesting.setTVSAllocation(0, total, 1, kols, amounts);

        // Claim the TVS as the EOA claimer (within claim window)
        vm.prank(claimer);
        vesting.claimRewardTVS(0);

        // Determine the minted NFT id (last minted id == totalMinted - 1)
        uint256 nftId = nft.getTotalMinted();

        // Prepare split percentages that sum to BASIS_POINT (10000)
        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5000;
        percentages[1] = 5000;

        // Calling splitTVS as the NFT owner should revert due to `_computeSplitArrays` writing into uninitialized memory arrays
        vm.prank(claimer);
        vm.expectRevert();
        vesting.splitTVS(0, percentages, nftId);
    }
```

### Mitigation

- Allocate the arrays inside `alloc` before writing to them.
  