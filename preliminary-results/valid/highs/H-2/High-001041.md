# [001041] Users cannot merge or split TVS due to uninitialized array causing DoS in FeeManager::calculateFeeAndNewAmountForOneTVS
  
  ## Summary

The uninitialized `newAmounts` array in `FeesManager.calculateFeeAndNewAmountForOneTVS()` will cause all merge and split operations to revert with array out-of-bounds error, preventing users from consolidating or dividing their TVS allocations.

## Root Cause
https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L175
In `FeesManager.sol:169-174` the `newAmounts` array is declared as a return parameter but never initialized, resulting in a zero-length array that causes out-of-bounds access.

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
    public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    // ❌ newAmounts is never initialized - defaults to empty array with length 0
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;  // ❌ OUT OF BOUNDS: Writing to index i of empty array
    }
}
```

When a memory array is declared but not initialized in Solidity, it defaults to an empty array with `length = 0`. Any attempt to write to `newAmounts[0]` or higher indices will revert with array out-of-bounds panic (0x32).

## Internal Pre-conditions

1. User needs to have at least 2 TVS NFTs to merge, OR 1 TVS NFT to split
2. Protocol owner needs to set merge or split fee rate to any value (including 0)
3. User attempts to call `mergeTVS()` or `splitTVS()`

## External Pre-conditions

None - this is a deterministic code bug that always triggers.

## Attack Path

This is a vulnerability path that completely breaks merge and split functionality:

1. **User receives or claims multiple TVS NFTs** (e.g., 3 NFTs with token allocations)
2. **User calls `mergeTVS(projectId, nftId, projectIds, [nft1, nft2])`** to consolidate allocations
3. **Contract executes `AlignerzVesting.sol:1013`**: calls `calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows)`
4. **`FeesManager` attempts to write to `newAmounts[0]`** in the loop
5. **Transaction reverts with panic 0x32** (array out-of-bounds access)
6. **User cannot merge TVS**

Same path applies to `splitTVS()` at line 1069.

## Impact

All users are completely unable to merge or split their TVS allocations. This breaks core protocol functionality and locks users into their initial allocation structure. Users cannot optimize their positions, consolidate for trading, or divide for partial sales.

Affected functions:
- **`mergeTVS()` at AlignerzVesting.sol:1002** - 100% broken
- **`splitTVS()` at AlignerzVesting.sol:1054** - 100% broken

This is a **complete denial of service** for two critical protocol features that manage TVS flexibility.

## PoC
Add this test to `test/AlignerzVestingProtocolTest.t.sol` 
- We expect the test to revert due to out of bound

```solidity
   function test_uninitializationBug() public {
        vm.startPrank(projectCreator);
        vesting.setMergeFeeRate(200); // 2% fee

        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 1000 ether;
        amounts[1] = 1000 ether;
        amounts[2] = 1000 ether;

        uint256 mergeRate = vesting.mergeFeeRate();
        // This function is public so we can call it directly for testing purposes
        // We expect this function to revert since `newAmounts` is not initialized
        vm.expectRevert();
        (uint256 feeAmount, uint256[] memory newAmounts) = vesting.calculateFeeAndNewAmountForOneTVS(
            mergeRate,
            amounts,
            amounts.length
        );

    }
```
### Same issue affects splitTVS():

1. Bob has 1 TVS NFT with 3000 tokens
2. Bob calls `splitTVS(0, [5000, 5000], nftId)` to split 50/50
3. Contract calls `calculateFeeAndNewAmountForOneTVS()` at line 1069
4. Transaction reverts with same panic
5. Bob cannot split his TVS

## Mitigation

Initialize the `newAmounts` array with proper length in `FeesManager.sol:169-174`:

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
    public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    newAmounts = new uint256[](length);  // ✓ Initialize array with correct length
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;  // ✓ Now safe to write
        unchecked {
            ++i;
        }
    }
}
```

  